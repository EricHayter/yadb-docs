---
layout: post
title:  "dev/1: Page Buffer Manager and Asynchronous Disk I/O in YADB"
date:   2025-06-15 21:00:00
categories: dev
---

![estimation: a comic by xkcd]({{ site.baseurl }}/assets/images/estimation.png)

This is the dev post for the major milestone of completing the page buffer manager in YADB! I'll start by going over the motivation behind having a page buffer (for more details, check out my [last post]({% post_url 2025-06-02-page-buffer %})) and then walk through each major component that makes up the page buffer subsystem as implemented in YADB. Below is an architecture diagram showing how the page buffer manager interacts with other subsystems.

We'll begin with the motivation and then build our way up from the foundational components to the page buffer manager itself.

![architecture diagram of the page buffer manager]({{ site.baseurl }}/assets/images/page_buffer_manager_architecture.png)

## Motivation

As mentioned in my [last post]({% post_url 2025-06-02-page-buffer %}), page buffers provide an abstraction that enables faster reads and writes to data originating from disk. The page buffer temporarily caches pages in memory, reducing the need for repeated disk I/O.

## Disk Manager

The disk manager is responsible for managing the database file stored on disk. The format is simple: a file composed of 4KB pages, where each page is identified by a page ID calculated from its byte offset (e.g., the page at offset 0 is page 0; the page at offset 65536 = 4KB * 16 is page 16). The disk manager supports dynamic growth but does not shrink the file when pages are deleted. Instead, deleted pages are repurposed for future use.

```cpp
// File: include/storage/disk/disk_manager.h
class DiskManager {
public:
    DiskManager(const std::filesystem::path& db_directory);
    DiskManager(const std::filesystem::path& db_directory, std::size_t page_capacity);
    ~DiskManager();

    page_id_t AllocatePage();
    bool WritePage(page_id_t page_id, PageView page_data);
    bool ReadPage(page_id_t page_id, MutPageView page_data);
    void DeletePage(page_id_t page_id);

private:
    static constexpr std::string_view DB_FILE_NAME = "data.db";
    static constexpr std::string_view DISK_MANAGER_LOG_FILE_NAME = "disk_manager.log";

    std::size_t GetOffset(page_id_t page_id);
    std::size_t GetDatabaseFileSize();

    std::size_t page_capacity_m;

    // list of pages that are considered free
    std::unordered_set<page_id_t> free_pages_m;

    std::fstream db_io_m;
    std::filesystem::path db_directory_m;
    std::shared_ptr<spdlog::logger> logger_m;
};
```

## Disk Scheduler

The disk scheduler enables asynchronous disk I/O. Waiting on disk operations serially is costly, so the scheduler improves throughput by processing requests in a dedicated background thread.

It uses a thread-safe queue and a worker thread to process queued tasks. These tasks include page allocation, deletion, read, and write operations, each mapped directly to a disk manager function:

```cpp
// File: include/storage/disk/disk_scheduler.h
class DiskScheduler {
public:
    DiskScheduler(const std::filesystem::path& db_file);
    ~DiskScheduler();

    void AllocatePage(std::promise<page_id_t>&& result);
    void DeletePage(page_id_t page_id, std::promise<void>&& done);
    void ReadPage(page_id_t page_id, MutPageView data, std::promise<bool>&& status);
    void WritePage(page_id_t page_id, PageView data, std::promise<bool>&& status);

private:
    void WorkerFunction(std::stop_token stop_token);
    std::jthread worker_thread_m;
    std::condition_variable cv_m;
    std::mutex mut_m;
    std::queue<IOTasks::Task> tasks_m;
    DiskManager disk_manager_m;
};
```

Each task is represented as a `std::variant` of specific structs, encapsulating the parameters and a `std::promise` for the result:

```cpp
// File: include/storage/disk/io_tasks.h
namespace IOTasks {

struct AllocatePageTask {
    std::promise<page_id_t> result;
};

struct DeletePageTask {
    page_id_t page_id;
    std::promise<void> done;
};

struct WritePageTask {
    page_id_t page_id;
    PageView data;
    std::promise<bool> status;
};

struct ReadPageTask {
    page_id_t page_id;
    MutPageView data;
    std::promise<bool> status;
};

using Task = std::variant<AllocatePageTask, DeletePageTask, WritePageTask, ReadPageTask>;

}; // namespace IOTasks
```

The worker thread processes these using `std::visit`:

```cpp
// File: src/storage/disk/disk_scheduler.cpp
void DiskScheduler::WorkerFunction(std::stop_token stop_token)
{
    while (not stop_token.stop_requested()) {
        std::unique_lock<std::mutex> lk(mut_m);
        cv_m.wait(lk, [this, &stop_token]() {
            return not tasks_m.empty() || stop_token.stop_requested();
        });

        if (stop_token.stop_requested())
            break;

        while (not tasks_m.empty()) {
            IOTasks::Task task = std::move(tasks_m.front());
            tasks_m.pop();
            lk.unlock();

            std::visit([this](auto&& task) {
                using T = std::decay_t<decltype(task)>;
                if constexpr (std::is_same_v<T, IOTasks::AllocatePageTask>) {
                    page_id_t page_id = disk_manager_m.AllocatePage();
                    task.result.set_value(page_id);
                } else if constexpr (std::is_same_v<T, IOTasks::DeletePageTask>) {
                    disk_manager_m.DeletePage(task.page_id);
                    task.done.set_value();
                } else if constexpr (std::is_same_v<T, IOTasks::ReadPageTask>) {
                    task.status.set_value(disk_manager_m.ReadPage(task.page_id, task.data));
                } else if constexpr (std::is_same_v<T, IOTasks::WritePageTask>) {
                    task.status.set_value(disk_manager_m.WritePage(task.page_id, task.data));
                }
            }, std::move(task));

            lk.lock();
        }
    }
}
```

## Page Buffer Manager

(For background on how the page buffer works, check out my [last post]({% post_url 2025-06-02-page-buffer %}))

The page buffer manager exposes three primary operations: creating a page, acquiring a handle for read access, and acquiring a handle for write access.

```cpp
// File: include/buffer/page_buffer_manager.h
class PageBufferManager {
    friend ReadPageGuard;
    friend WritePageGuard;

public:
    PageBufferManager(const std::filesystem::path& db_directory, std::size_t num_frames);
    ~PageBufferManager();
    page_id_t NewPage();

    std::optional<ReadPageGuard> TryReadPage(page_id_t page_id);
    ReadPageGuard WaitReadPage(page_id_t page_id);

    std::optional<WritePageGuard> TryWritePage(page_id_t page_id);
    WritePageGuard WaitWritePage(page_id_t page_id);

private:
    bool LoadPage(page_id_t page_id);
    bool FlushPage(page_id_t page_id);
    void AddAccessor(frame_id_t frame_id, bool is_writer);
    void RemoveAccessor(frame_id_t frame_id);

private:
    static constexpr std::string_view PAGE_BUFFER_MANAGER_LOG_FILENAME{ "page_buffer_manager.log" };
    std::shared_ptr<spdlog::logger> logger_m;

    LRUKReplacer replacer_m;
    DiskScheduler disk_scheduler_m;

    char* buffer_m;

    std::unordered_map<page_id_t, frame_id_t> page_map_m;
    std::vector<std::unique_ptr<FrameHeader>> frames_m;

    std::mutex mut_m;
    std::condition_variable available_frame_m;
};
```

These operations return RAII guard types which enforce access rules: multiple readers or a single writer. They use `shared_mutex` to manage access to each page.

```cpp
// File: include/buffer/page_guard.h
class PageBufferManager;

class ReadPageGuard {
public:
    ReadPageGuard(PageBufferManager* page_buffer_manager, FrameHeader* frame_header, std::shared_lock<std::shared_mutex>&& lk);
    ~ReadPageGuard();
    ReadPageGuard(ReadPageGuard&& other);
    ReadPageGuard& operator=(ReadPageGuard&& other);

    PageView GetData();

private:
    PageBufferManager* page_buffer_manager_m;
    FrameHeader* frame_header_m;
    std::shared_lock<std::shared_mutex> lk_m;
};

class WritePageGuard {
public:
    WritePageGuard(PageBufferManager* page_buffer_manager, FrameHeader* frame_header, std::unique_lock<std::shared_mutex>&& lk);
    ~WritePageGuard();
    WritePageGuard(WritePageGuard&& other);
    WritePageGuard& operator=(WritePageGuard&& other);

    MutPageView GetData();

private:
    PageBufferManager* page_buffer_manager_m;
    FrameHeader* frame_header_m;
    std::unique_lock<std::shared_mutex> lk_m;
};
```

These guards also notify the page buffer manager when they are created and destroyed. This allows the manager to maintain usage counts for eviction decisions.

Example: the constructor and destructor for `ReadPageGuard`:

```cpp
// File: src/buffer/page_guard.cpp
ReadPageGuard::ReadPageGuard(PageBufferManager* page_buffer_manager, FrameHeader* frame_header, std::shared_lock<std::shared_mutex>&& lk)
    : page_buffer_manager_m(page_buffer_manager)
    , frame_header_m(frame_header)
    , lk_m(std::move(lk))
{
    assert(page_buffer_manager != nullptr);
    assert(frame_header != nullptr);

    // Creating a page guard is contingent on owning a lock to the frame's
    // shared lock. Otherwise, thread safety cannot be guaranteed, undermining
    // the job the guard itself.
    assert(lk_m.owns_lock());
    page_buffer_manager_m->AddAccessor(frame_header_m->id, false);
}

ReadPageGuard::~ReadPageGuard()
{
    if (page_buffer_manager_m != nullptr)
        page_buffer_manager_m->RemoveAccessor(frame_header_m->id);
}

```

This approach of using views into the buffer may seem unintuitive compared to just passing around `span` or `vector` objects. However, this design enables **zero-copy** access—clients can write directly to the buffer without needing to allocate and copy data from intermediate structures.

## Conclusion

With the implementation of the disk manager, disk scheduler, and page buffer manager, YADB now has a complete subsystem for efficient, asynchronous page-level disk I/O. These components form the backbone for transaction-safe storage in the database engine and are designed to scale well with concurrent workloads. Next up will be integrating the page buffer manager with concurrency control and logging—stay tuned!


