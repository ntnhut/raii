# Advanced RAII Techniques

Once you’ve mastered the fundamentals, RAII becomes a powerful design tool for building reliable, maintainable, and exception-safe C++ systems.

This section explores how to design custom RAII classes using **move semantics**, **custom deleters**, **polymorphism**, and **modern concurrency features**, and how to apply RAII to **higher-level abstractions** like transactions or thread pools.

## Designing RAII classes with move semantics

A good RAII class clearly expresses **ownership** — one object is responsible for acquiring and releasing a resource.

In modern C++, **move semantics** make that ownership transfer safe and efficient.

### Example: A simple RAII file wrapper

```cpp
#include <cstdio>
#include <stdexcept>

class FileHandle {
    FILE* file = nullptr;

public:
    explicit FileHandle(const char* filename, const char* mode) {
        file = std::fopen(filename, mode);
        if (!file)
            throw std::runtime_error("Failed to open file");
    }

    // Non-copyable
    FileHandle(const FileHandle&) = delete;
    FileHandle& operator=(const FileHandle&) = delete;

    // Movable
    FileHandle(FileHandle&& other) noexcept : file(other.file) {
        other.file = nullptr;
    }

    FileHandle& operator=(FileHandle&& other) noexcept {
        if (this != &other) {
            close();
            file = other.file;
            other.file = nullptr;
        }
        return *this;
    }

    ~FileHandle() { close(); }

    void close() {
        if (file) {
            std::fclose(file);
            file = nullptr;
        }
    }

    FILE* get() const { return file; }
};
```

Usage example:

```cpp
int main() {
    FileHandle f1("data.txt", "w");
    std::fputs("Hello RAII\n", f1.get());

    FileHandle f2 = std::move(f1); // ownership transferred
} // file closed automatically here
```

RAII ensures deterministic cleanup even during exceptions or early returns.
 
## Custom Deleters for Flexible Cleanup

C++ RAII integrates beautifully with C-style APIs that use manual allocation/freeing functions.
With `std::unique_ptr`’s **custom deleters**, you can automatically manage any C resource.

### Example: Managing a C API Resource

```cpp
#include <iostream>
#include <memory>

// --- Simulated C API ---
struct CResource {
    int id;
};

CResource* create_resource() {
    static int counter = 0;
    auto* res = new CResource{++counter};
    std::cout << "CResource " << res->id << " created\n";
    return res;
}

void destroy_resource(CResource* res) {
    std::cout << "CResource " << res->id << " destroyed\n";
    delete res;
}

// --- RAII Usage ---
using ResourcePtr = std::unique_ptr<CResource, void(*)(CResource*)>;

void use_resource(bool fail = false) {
    ResourcePtr res(create_resource(), destroy_resource);

    std::cout << "Using CResource " << res->id << "\n";
    if (fail)
        throw std::runtime_error("Operation failed!");
} // destroy_resource() automatically called
```

Try calling:

```cpp
int main() {
    try {
        use_resource(true);
    } catch (...) {
        std::cout << "Caught exception safely.\n";
    }
}
```

Output:

```text
CResource 1 created
Using CResource 1
CResource 1 destroyed
Caught exception safely.
```

Even with an exception, the cleanup happens automatically — exactly what RAII guarantees.
 

## RAII with polymorphic cleanup

Sometimes, you want different resource types managed through a single interface.

By combining RAII with **virtual destructors**, you can ensure each subclass cleans up correctly.

### Example: Polymorphic resource base

```cpp
#include <iostream>
#include <memory>

struct ResourceBase {
    virtual ~ResourceBase() = default;
    virtual void release() = 0;
};

struct FileResource : ResourceBase {
    ~FileResource() override { release(); }
    void release() override { std::cout << "File closed\n"; }
};

struct NetworkResource : ResourceBase {
    ~NetworkResource() override { release(); }
    void release() override { std::cout << "Connection terminated\n"; }
};

void use_polymorphic_raii() {
    std::unique_ptr<ResourceBase> res = std::make_unique<FileResource>();
} // Correct release() called automatically
```

RAII ensures the right destructor and cleanup logic are invoked — even when managed via base pointers.
 

## RAII and concurrency primitives

Many concurrency-related resources (locks, threads) must be released deterministically.

The standard library provides several RAII wrappers for them.

### Example: Locking with `std::lock_guard`

```cpp
#include <mutex>

std::mutex m;

void safe_increment(int& counter) {
    std::lock_guard<std::mutex> lock(m);
    ++counter;
} // lock released automatically here
```

Even if an exception is thrown inside, the mutex unlocks automatically — no need for manual `unlock()`.
 
### Example: Thread management with `std::jthread`

C++20 introduced `std::jthread`, which automatically joins on destruction.

```cpp
#include <thread>
#include <iostream>

void run_task() {
    std::cout << "Running in thread\n";
}

void start_thread() {
    std::jthread t(run_task);
    std::cout << "Main continues\n";
} // t joins automatically here
```

RAII ensures threads always complete cleanly, even with exceptions or early returns.
 
## RAII for higher-Level abstractions

RAII isn’t limited to low-level resources — it can model any *acquire/release* lifecycle, such as database transactions or thread pools.
 

### Example: Scoped transaction

```cpp
#include <iostream>
#include <stdexcept>

struct Transaction {
    bool committed = false;

    void commit() {
        committed = true;
        std::cout << "Transaction committed\n";
    }

    ~Transaction() {
        if (!committed)
            std::cout << "Transaction rolled back\n";
    }
};

void process(bool fail) {
    Transaction txn;
    std::cout << "Transaction started\n";

    // Simulate some work
    if (fail)
        throw std::runtime_error("Something went wrong!");

    txn.commit(); // Mark as successful
}

int main() {
    try {
        process(true);
    } catch (const std::exception& e) {
        std::cout << "Caught: " << e.what() << "\n";
    }

    process(false);
}
```

Output:

```text
Transaction started
Transaction rolled back
Caught: Something went wrong!
Transaction started
Transaction committed
```

RAII guarantees rollback on exceptions or early returns — no manual cleanup or error checking needed.
 

### Example: Thread pool scope guard

Let’s consider a mini thread pool that starts worker threads and ensures they shut down safely.

```cpp
#include <iostream>
#include <thread>
#include <vector>
#include <atomic>
#include <chrono>

class ThreadPool {
    std::vector<std::thread> workers;
    std::atomic<bool> running{true};

public:
    ThreadPool(size_t num_threads = 2) {
        std::cout << "Starting thread pool with " << num_threads << " threads\n";
        for (size_t i = 0; i < num_threads; ++i) {
            workers.emplace_back([this, i] {
                while (running) {
                    std::this_thread::sleep_for(std::chrono::milliseconds(200));
                    std::cout << "Worker " << i << " running\n";
                }
                std::cout << "Worker " << i << " exiting\n";
            });
        }
    }

    ~ThreadPool() {
        std::cout << "Shutting down thread pool...\n";
        running = false;
        for (auto& t : workers)
            if (t.joinable())
                t.join();
    }
};

int main() {
    {
        ThreadPool pool(2);
        std::this_thread::sleep_for(std::chrono::milliseconds(500));
    } // Pool automatically stops here
    std::cout << "Main exiting\n";
}
```

Output:

```text

Starting thread pool with 2 threads
Worker 1 running
Worker 0 running
Worker 0Worker 1 running
 running
Shutting down thread pool...
Worker 1 running
Worker 1 exiting
Worker 0 running
Worker 0 exiting
Main exiting
```

RAII ensures all worker threads are joined automatically — no leaks, no dangling threads, no forgotten cleanup.
 
## Summary

Advanced RAII design builds upon the same foundation — **tie resource lifetime to object lifetime** — but extends it to richer domains:

* Move semantics enable **unique, transferable ownership**.
* Custom deleters make cleanup **flexible and compatible with C APIs**.
* Virtual destructors provide **polymorphic resource handling**.
* Concurrency primitives gain **automatic safety guarantees**.
* High-level systems like **transactions and thread pools** benefit from **deterministic cleanup**.

RAII doesn’t just prevent leaks — it simplifies complex systems by guaranteeing that *no matter what happens, cleanup always occurs correctly.*
 
