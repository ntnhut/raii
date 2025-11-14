# RAII In Practice: Patterns And Examples

By now, you've seen what RAII is and why it matters. Let's see it in action — both in the C++ Standard Library and in custom user-defined classes.

RAII is everywhere in C++: memory containers, streams, smart pointers, locks, and even high-performance systems code. You use it every day, often without realizing it.

This chapter explores the most common RAII patterns and idioms you'll encounter (and use) throughout your C++ career.

## Memory management with containers

Dynamic memory is one of the hardest resources to manage correctly.

Before C++11, forgetting a `delete` was a frequent source of leaks and crashes.

Containers like `std::vector` and `std::string` solve this problem by managing heap memory automatically — perfect examples of RAII in practice.

### Example: `std::vector`

```cpp
#include <vector>

void exampleVector() {
    std::vector<int> data = {1, 2, 3, 4, 5};  // memory acquired
    data.push_back(6);
}  // vector destroyed → memory automatically released
```

When `data` goes out of scope, its destructor automatically releases the allocated memory. No manual `delete[]` required.

Behind the scenes:

* Constructor: allocates heap memory.
* Destructor: frees memory when the vector is destroyed.
* Copy/move constructors manage ownership transfer.

This is classic RAII — safe and automatic memory management.

### Example: `std::string`

```cpp
#include <string>
#include <iostream>

void exampleString() {
    std::string name = "RAII example";  // allocates dynamic memory
    std::cout << name << "\n";
}  // memory released automatically
```

`std::string` manages a heap buffer internally.

If you assign a longer string, it reallocates; when it's destroyed, it releases the memory.

No leaks, no dangling pointers — all managed by the destructor.

## The scoped resource pattern

The scoped resource pattern is the essence of RAII.

It means: **a resource is valid for the lifetime of an object, and automatically released when that object goes out of scope.**

### File handles: safe and automatic

Managing file handles manually often leads to leaks when exceptions or early returns happen.

`std::ifstream`, `std::ofstream`, and `std::fstream` are RAII wrappers around file descriptors.

```cpp
#include <fstream>

void writeLog() {
    std::ofstream file("log.txt");  // open file
    if (!file) throw std::runtime_error("Failed to open file");
    
    file << "Application started\n";
    file << "Logging data...\n";
}  // file closed automatically
```

When `file` goes out of scope, its destructor closes the file descriptor — even if an exception is thrown.

This makes file I/O exception-safe and leak-free.

### A Complete RAII File Wrapper

Let's build a complete, production-quality RAII wrapper for file handles. 
This example will demonstrate all the key principles we've discussed, and 
we'll reference it throughout the rest of this book when exploring advanced 
topics like move semantics, testing, and debugging.

```cpp
#include <cstdio>
#include <stdexcept>
#include <utility>

class FileHandle {
    FILE* file_ = nullptr;

public:
    // Constructor acquires the resource
    explicit FileHandle(const char* path, const char* mode) 
        : file_(std::fopen(path, mode)) {
        if (!file_) {
            throw std::runtime_error("Failed to open file");
        }
    }

    // Destructor releases the resource
    ~FileHandle() noexcept {
        if (file_) {
            std::fclose(file_);
            file_ = nullptr;
        }
    }

    // Delete copy operations - file handles shouldn't be copied
    FileHandle(const FileHandle&) = delete;
    FileHandle& operator=(const FileHandle&) = delete;

    // Implement move operations - ownership can be transferred
    FileHandle(FileHandle&& other) noexcept 
        : file_(other.file_) {
        other.file_ = nullptr;
    }

    FileHandle& operator=(FileHandle&& other) noexcept {
        if (this != &other) {
            // Close our current file if we have one
            if (file_) {
                std::fclose(file_);
            }
            // Take ownership of other's file
            file_ = other.file_;
            other.file_ = nullptr;
        }
        return *this;
    }

    // Provide safe access to the underlying handle
    FILE* get() const noexcept { return file_; }
    
    // Check if file is open
    bool is_open() const noexcept { return file_ != nullptr; }
};
```
This implementation follows the Rule of Five, which states that if you 
explicitly define any of the five special member functions (destructor, copy 
constructor, copy assignment, move constructor, move assignment), you should 
carefully consider all five.

The destructor is marked noexcept because destructors should never throw 
exceptions. During stack unwinding from an exception, if a destructor throws, 
the program will call std::terminate and crash immediately.

We delete the copy constructor and copy assignment operator because copying 
a file handle doesn't make semantic sense. If you copied the raw FILE pointer, 
both objects would try to close the same file when destroyed, causing undefined 
behavior. This is called the double-free problem.

However, we do implement move operations. Moving means transferring ownership 
from one object to another. After a move, the source object no longer owns the 
file handle, so it won't try to close it. This is safe and often necessary when 
returning `FileHandle` objects from functions or storing them in containers.

Usages:

```cpp
void example_basic_usage() {
    FileHandle file("data.txt", "w");
    std::fprintf(file.get(), "Hello, RAII!\n");
} // file automatically closed here

void example_early_return(bool condition) {
    FileHandle file("log.txt", "a");
    
    if (condition) {
        std::fprintf(file.get(), "Early exit\n");
        return; // file still gets closed automatically
    }
    
    std::fprintf(file.get(), "Normal path\n");
} // or here

void example_exception_safety() {
    FileHandle file("config.txt", "r");
    
    // Even if this throws an exception...
    parse_complex_data(file.get());
    
} // ...the file is guaranteed to close
```

This `FileHandle` class demonstrates all the RAII principles we've learned: 
resource acquisition in the constructor, guaranteed cleanup in the destructor, 
clear ownership semantics through deleted copy and implemented move operations, 
and automatic exception safety. Throughout the rest of this book, when we refer 
to `FileHandle`, we mean this implementation.

## Lock guards and scoped locks

Another widespread RAII idiom is automatic lock management.

Instead of manually locking and unlocking a mutex, you wrap the lock in an object that unlocks automatically when it goes out of scope.

### Mutex locks and synchronization

Threads require synchronization to avoid race conditions.

Manually calling `lock()` and `unlock()` can easily cause deadlocks if the unlock call is skipped.

RAII solves this neatly with `std::lock_guard` and `std::unique_lock`.

```cpp
#include <mutex>
#include <thread>
#include <iostream>

std::mutex m;

void printMessage() {
    std::lock_guard<std::mutex> lock(m);  // lock acquired here
    std::cout << "Hello from thread " << std::this_thread::get_id() << '\n';
}  // lock automatically released here
```

No matter how the function exits (normal return or exception), the mutex will always be unlocked.

This makes multithreaded code safe by default.

### Example: Lock management with RAII

```cpp
#include <mutex>

std::mutex mtx;

void safeIncrement(int& counter) {
    std::lock_guard<std::mutex> lock(mtx);  // lock acquired
    ++counter;
}  // lock released automatically here
```

Even if an exception occurs inside the critical section, the lock is released automatically when `lock_guard` is destroyed.

### Multiple mutex locking with `std::scoped_lock`

C++17 introduced `std::scoped_lock`, which can manage multiple mutexes safely and avoid deadlocks:

```cpp
std::mutex m1, m2;

void safeFunction() {
    std::scoped_lock lock(m1, m2);  // locks both atomically
    // critical section
}
```

### More flexible locks with `std::unique_lock`

`std::unique_lock` provides more flexibility than `std::lock_guard`, such as deferred locking or manual unlock.

```cpp
void exampleUniqueLock() {
    std::unique_lock<std::mutex> lock(mtx, std::defer_lock);
    // do some preparation work
    
    lock.lock();  // explicitly acquire lock
    // work with shared data
    lock.unlock();  // manually release before end of scope
}
```

The destructor still ensures cleanup if you forget to unlock manually — RAII has your back.

## Smart pointers: RAII for dynamic memory

Before C++11, developers had to manage heap memory manually with `new` and `delete`.

Now, smart pointers like `std::unique_ptr` and `std::shared_ptr` encapsulate ownership semantics and automatic destruction.

### Example: `std::unique_ptr`

`std::unique_ptr` represents exclusive ownership.

When the smart pointer is destroyed, it deletes the managed object.

```cpp
#include <memory>
#include <iostream>

void exampleUniquePtr() {
    auto ptr = std::make_unique<int>(42);
    std::cout << *ptr << "\n";
}  // memory automatically deleted
```

No `delete` needed — memory cleanup is tied to the lifetime of `ptr`.

You can also transfer ownership:

```cpp
auto a = std::make_unique<int>(10);
auto b = std::move(a);  // ownership transferred
```

### Example: `std::shared_ptr`

`std::shared_ptr` allows shared ownership of a resource.

It uses a reference count to track how many owners exist.

When the last owner goes out of scope, the resource is released.

```cpp
#include <memory>
#include <iostream>

void exampleSharedPtr() {
    std::shared_ptr<int> p1 = std::make_shared<int>(5);
    {
        std::shared_ptr<int> p2 = p1;  // shared ownership
        std::cout << "Count: " << p1.use_count() << "\n";
    }  // p2 destroyed, count decremented
    std::cout << "Count: " << p1.use_count() << "\n";
}  // p1 destroyed → memory deleted
```

Output:
```
Count: 2
Count: 1
```

RAII ensures correct destruction only when no one is using the resource anymore.

## Temporary state and scoped guards

Sometimes, RAII is used not for an external resource but for temporary state changes. You set up a new state in the constructor and restore it in the destructor.

### Example: Scoped flag

```cpp
class ScopedFlag {
    bool& flag;
public:
    ScopedFlag(bool& f) : flag(f) { flag = true; }
    ~ScopedFlag() { flag = false; }
};
```

Usage:

```cpp
bool isProcessing = false;

void doWork() {
    ScopedFlag guard(isProcessing);
    // isProcessing = true during this scope
    // do some work
}  // isProcessing reset to false
```

This pattern is often used in graphics code, GUI frameworks, or testing environments where temporary global states must be restored.

## Scoped timer

Timers are another practical example of the scoped resource pattern.

You can measure execution time automatically by starting a timer in the constructor and stopping it in the destructor.

### Custom RAII class: ScopedTimer

```cpp
#include <chrono>
#include <iostream>
#include <string>

class ScopedTimer {
    std::string name;
    std::chrono::steady_clock::time_point start;
public:
    explicit ScopedTimer(std::string n)
        : name(std::move(n)), start(std::chrono::steady_clock::now()) {
        std::cout << name << " started.\n";
    }
    
    ~ScopedTimer() {
        auto end = std::chrono::steady_clock::now();
        auto ms = std::chrono::duration_cast<std::chrono::milliseconds>(end - start).count();
        std::cout << name << " finished in " << ms << " ms.\n";
    }
};
```

Usage:

```cpp
void heavyComputation() {
    ScopedTimer timer("Computation");
    // simulate work
    std::this_thread::sleep_for(std::chrono::milliseconds(200));
}
```

Output:
```
Computation started.
Computation finished in 200 ms.
```

Even if an exception is thrown or the function returns early, the timer's destructor still logs the duration.

This is a perfect example of applying RAII to logic beyond memory or files.

This pattern shows how RAII can simplify instrumentation and profiling logic — the destructor runs automatically when the scope ends.

## Custom scoped cleanup (Defer idiom)

Sometimes you just need to run a specific cleanup action when a scope ends.

You can build your own "defer" utility:

```cpp
#include <functional>

class ScopeExit {
    std::function<void()> func;
public:
    ScopeExit(std::function<void()> f) : func(std::move(f)) {}
    ~ScopeExit() { func(); }
};
```

Usage:

```cpp
void example() {
    ScopeExit on_exit([&]() { std::cout << "Goodbye!\n"; });
    std::cout << "Hello!\n";
}  // prints "Hello!" then "Goodbye!"
```

This pattern has become quite popular in modern C++ because it lets you express intent very locally — similar to Go's `defer`.

## Other practical applications

RAII isn't limited to standard library classes.

It's used for:

* Managing database connections (open in constructor, close in destructor).
* Handling sockets or network buffers.
* Registering/unregistering callbacks.
* Acquiring GPU or system handles.
* Automatically restoring global states (e.g., current directory, thread affinity).

Anywhere there's a "get–use–release" pattern, RAII fits.

## Summary: Common RAII patterns

Let's recap the patterns we've explored:

| Pattern | Typical Use | Resource Released In |
|---------|-------------|---------------------|
| *ScopedFile* | File I/O | Destructor |
| *std::lock_guard / std::scoped_lock* | Mutex locking | Destructor |
| *ScopedFlag* | Temporary state change | Destructor |
| *ScopedTimer* | Timing / profiling | Destructor |
| *Smart pointers* (unique_ptr, shared_ptr) | Memory ownership | Destructor |
| *ScopeExit / "defer" idiom* | Arbitrary cleanup | Destructor |

And a summary by resource type:

| Resource Type | RAII Example | Automatically Released By |
|--------------|--------------|--------------------------|
| *Dynamic memory* | `std::vector`, `std::unique_ptr` | Destructor |
| *File handles* | `std::ofstream`, `FileHandle` | Destructor |
| *Locks* | `std::lock_guard`, `std::unique_lock` | Destructor |
| *Shared memory* | `std::shared_ptr` | Reference counting |
| *Timers/loggers* | `ScopedTimer`, custom RAII | Destructor |

The unifying idea: **RAII ensures deterministic cleanup tied to scope, making your code both simpler and safer.**

RAII makes C++ code safe, clean, and maintainable.

You don't have to manually track cleanup — it's guaranteed by the language itself.
