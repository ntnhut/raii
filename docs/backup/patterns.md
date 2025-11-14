# Common RAII Patterns and Idioms

RAII is more than just smart pointers — it’s a *design principle* that shows up across the entire C++ ecosystem. Many standard and custom types are built around the same idea: acquire a resource in a constructor, and release it in the destructor.

Let’s go through some of the most common RAII patterns and idioms that you’ll encounter (and often use without realizing).

 
## Scoped resource pattern

The *scoped resource* pattern is the essence of RAII.

It means: *a resource is valid for the lifetime of an object, and automatically released when that object goes out of scope.*

A simple example is a file wrapper:

```cpp
#include <cstdio>

class ScopedFile {
    FILE* file;
public:
    ScopedFile(const char* path, const char* mode)
        : file(std::fopen(path, mode)) {}

    ~ScopedFile() {
        if (file)
            std::fclose(file);
    }

    FILE* get() const { return file; }

    // disallow copy, allow move
    ScopedFile(const ScopedFile&) = delete;
    ScopedFile& operator=(const ScopedFile&) = delete;
    ScopedFile(ScopedFile&& other) noexcept : file(other.file) {
        other.file = nullptr;
    }
};
```

Usage:

```cpp
void writeLog() {
    ScopedFile log("log.txt", "w");
    std::fprintf(log.get(), "Program started\n");
} // fclose automatically called
```

This is the simplest but most important RAII pattern.

It’s about *scoping*: the resource’s lifetime matches the scope of the managing object.

You’ll find this pattern in many standard library utilities like:

* `std::lock_guard` – for scoped mutex locking.
* `std::ofstream` – for scoped file output.
* `std::unique_ptr` – for scoped heap ownership.

 
## Lock guards and scoped locks

Another widespread RAII idiom is **automatic lock management**.

Instead of manually locking and unlocking a mutex, you wrap the lock in an object that unlocks automatically when it goes out of scope:

```cpp
#include <mutex>
#include <thread>
#include <iostream>

std::mutex m;

void printMessage() {
    std::lock_guard<std::mutex> lock(m); // lock acquired here
    std::cout << "Hello from thread " << std::this_thread::get_id() << '\n';
} // lock automatically released here
```

No matter how the function exits (normal return or exception), the mutex will always be unlocked.

This makes multithreaded code *safe by default*.

C++17 introduced `std::scoped_lock`, which can manage multiple mutexes safely and avoid deadlocks:

```cpp
std::mutex m1, m2;

void safeFunction() {
    std::scoped_lock lock(m1, m2); // locks both atomically
    // critical section
}
```

 
## Temporary state and scoped guards

Sometimes, RAII is used not for an external resource but for *temporary state changes*.
You set up a new state in the constructor and restore it in the destructor.

Example — a class that temporarily changes a flag:

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
} // isProcessing reset to false
```

This pattern is often used in graphics code, GUI frameworks, or testing environments where temporary global states must be restored.

 
## Scoped timer

Timers are another practical example of the scoped resource pattern.

You can measure execution time automatically by starting a timer in the constructor and stopping it in the destructor.

```cpp
#include <chrono>
#include <iostream>
#include <string>

class ScopedTimer {
    std::string label;
    std::chrono::high_resolution_clock::time_point start;
public:
    ScopedTimer(std::string name)
        : label(std::move(name)), start(std::chrono::high_resolution_clock::now()) {}

    ~ScopedTimer() {
        using namespace std::chrono;
        auto end = high_resolution_clock::now();
        auto duration = duration_cast<milliseconds>(end - start).count();
        std::cout << label << " took " << duration << " ms\n";
    }
};
```

Usage:

```cpp
void heavyComputation() {
    ScopedTimer timer("heavyComputation");
    // do some work
} // prints "heavyComputation took X ms"
```

This pattern shows how RAII can simplify instrumentation and profiling logic — the destructor runs automatically when the scope ends.

 
## Smart pointers: unique and shared ownership

Smart pointers are probably the most famous RAII idiom.

* `std::unique_ptr` manages *exclusive ownership* of a dynamically allocated object.
* `std::shared_ptr` manages *shared ownership* using reference counting.

Example:

```cpp
#include <memory>
#include <iostream>

void example() {
    std::unique_ptr<int> ptr = std::make_unique<int>(42);
    std::cout << *ptr << '\n';  // use resource
} // ptr deleted automatically
```

No manual `delete` required.
This concept extends to other types like `std::shared_ptr` for shared ownership or `std::weak_ptr` for non-owning observers.

 
## Custom scoped cleanup (Defer idiom)

Sometimes you just need to run *a specific cleanup action* when a scope ends.

You can build your own “defer” utility:

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
} // prints "Hello!" then "Goodbye!"
```

This pattern has become quite popular in modern C++ because it lets you express intent very locally — similar to Go’s `defer`.

 
## Summary

RAII is a general principle, and these patterns are all its manifestations:

| Pattern                                     | Typical Use            | Resource Released In |
| ------------------------------------------- | ---------------------- | -------------------- |
| `ScopedFile`                                | File I/O               | Destructor           |
| `std::lock_guard` / `std::scoped_lock`      | Mutex locking          | Destructor           |
| `ScopedFlag`                                | Temporary state change | Destructor           |
| `ScopedTimer`                               | Timing / profiling     | Destructor           |
| Smart pointers (`unique_ptr`, `shared_ptr`) | Memory ownership       | Destructor           |
| `ScopeExit` / “defer” idiom                 | Arbitrary cleanup      | Destructor           |

The unifying idea: **RAII ensures deterministic cleanup tied to scope**, making your code both simpler and safer.

