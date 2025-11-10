# RAII in Practice: Examples

By now, you’ve seen what RAII is and why it matters.
Let’s see it in action — both in the C++ Standard Library and in custom user-defined classes.

RAII is everywhere in C++: memory containers, streams, smart pointers, locks, and even high-performance systems code.
You use it every day, often without realizing it.

 
## Memory management with containers

Dynamic memory is one of the hardest resources to manage correctly.

Before C++11, forgetting a `delete` was a frequent source of leaks and crashes.

Containers like `std::vector` and `std::string` solve this problem by managing heap memory automatically — perfect examples of RAII in practice.

 
### Example 1.1: `std::vector`

```cpp
#include <vector>

void exampleVector() {
    std::vector<int> data = {1, 2, 3, 4, 5}; // memory acquired
    data.push_back(6);
} // vector destroyed → memory automatically released
```

When `data` goes out of scope, its destructor automatically releases the allocated memory.
No manual `delete[]` required.

Behind the scenes:

* Constructor: allocates heap memory.
* Destructor: frees memory when the vector is destroyed.
* Copy/move constructors manage ownership transfer.

This is classic RAII — safe and automatic memory management.

 
### Example 1.2: `std::string`

```cpp
#include <string>
#include <iostream>

void exampleString() {
    std::string name = "RAII example"; // allocates dynamic memory
    std::cout << name << "\n";
} // memory released automatically
```

`std::string` manages a heap buffer internally.

If you assign a longer string, it reallocates; when it’s destroyed, it releases the memory.

No leaks, no dangling pointers — all managed by the destructor.

 
## File handles: safe and automatic

Managing file handles manually often leads to leaks when exceptions or early returns happen.
`std::ifstream`, `std::ofstream`, and `std::fstream` are RAII wrappers around file descriptors.

### Example 2.1: file stream RAII

```cpp
#include <fstream>

void writeLog() {
    std::ofstream file("log.txt"); // open file
    if (!file) throw std::runtime_error("Failed to open file");
    
    file << "Application started\n";
    file << "Logging data...\n";
} // file closed automatically
```

When `file` goes out of scope, its destructor closes the file descriptor — even if an exception is thrown.

This makes file I/O **exception-safe** and **leak-free**.
 
## Mutex locks and synchronization

Threads require synchronization to avoid race conditions.

Manually calling `lock()` and `unlock()` can easily cause deadlocks if the unlock call is skipped.

RAII solves this neatly with `std::lock_guard` and `std::unique_lock`.

### Example 3.1: lock management with RAII

```cpp
#include <mutex>

std::mutex mtx;

void safeIncrement(int& counter) {
    std::lock_guard<std::mutex> lock(mtx); // lock acquired
    ++counter;
} // lock released automatically here
```

Even if an exception occurs inside the critical section, the lock is released automatically when `lock_guard` is destroyed.

 
### Example 3.2: more flexible locks

`std::unique_lock` provides more flexibility than `std::lock_guard`, such as deferred locking or manual unlock.

```cpp
void exampleUniqueLock() {
    std::unique_lock<std::mutex> lock(mtx, std::defer_lock);
    // do some preparation work
    lock.lock();  // explicitly acquire lock
    // work with shared data
    lock.unlock(); // manually release before end of scope
}
```

The destructor still ensures cleanup if you forget to unlock manually — RAII has your back.

 
## Smart pointers: RAII for dynamic memory

Before C++11, developers had to manage heap memory manually with `new` and `delete`.

Now, smart pointers like `std::unique_ptr` and `std::shared_ptr` encapsulate ownership semantics and automatic destruction.

 
### Example 4.1: `std::unique_ptr`

`std::unique_ptr` represents exclusive ownership.

When the smart pointer is destroyed, it deletes the managed object.

```cpp
#include <memory>
#include <iostream>

void exampleUniquePtr() {
    auto ptr = std::make_unique<int>(42);
    std::cout << *ptr << "\n";
} // memory automatically deleted
```

No `delete` needed — memory cleanup is tied to the lifetime of `ptr`.

You can also transfer ownership:

```cpp
auto a = std::make_unique<int>(10);
auto b = std::move(a); // ownership transferred
```

 
### Example 4.2: `std::shared_ptr`

`std::shared_ptr` allows **shared ownership** of a resource.

It uses a reference count to track how many owners exist.

When the last owner goes out of scope, the resource is released.

```cpp
#include <memory>
#include <iostream>

void exampleSharedPtr() {
    std::shared_ptr<int> p1 = std::make_shared<int>(5);
    {
        std::shared_ptr<int> p2 = p1; // shared ownership
        std::cout << "Count: " << p1.use_count() << "\n";
    } // p2 destroyed, count decremented
    std::cout << "Count: " << p1.use_count() << "\n";
} // p1 destroyed → memory deleted
```

Output:

```text
Count: 2
Count: 1
```

RAII ensures correct destruction only when no one is using the resource anymore.

 
## Custom RAII class with ScopedTimer

You can create your own RAII utilities for non-traditional resources — like timing a scope or logging events.

### Example 5.1: `ScopedTimer`

```cpp
#include <chrono>
#include <iostream>

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

```text
Computation started.
Computation finished in 200 ms.
```

Even if an exception is thrown or the function returns early, the timer’s destructor still logs the duration.

This is a perfect example of applying RAII to logic beyond memory or files.

 
## Other practical applications

RAII isn’t limited to standard library classes.

It’s used for:

* Managing database connections (open in constructor, close in destructor).
* Handling sockets or network buffers.
* Registering/unregistering callbacks.
* Acquiring GPU or system handles.
* Automatically restoring global states (e.g., current directory, thread affinity).

Anywhere there’s a “get–use–release” pattern, RAII fits.

 
## Summary

Let’s recap what we’ve seen:

| Resource Type  | RAII Example                          | Automatically Released By |
| -------------- | ------------------------------------- | ------------------------- |
| Dynamic memory | `std::vector`, `std::unique_ptr`      | Destructor                |
| File handles   | `std::ofstream`, `FileHandle`         | Destructor                |
| Locks          | `std::lock_guard`, `std::unique_lock` | Destructor                |
| Shared memory  | `std::shared_ptr`                     | Reference counting        |
| Timers/loggers | `ScopedTimer`, custom RAII            | Destructor                |

RAII makes C++ code **safe, clean, and maintainable**.

You don’t have to manually track cleanup — it’s guaranteed by the language itself.

