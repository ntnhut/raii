
# Testing and Debugging RAII Code

RAII is designed to *prevent* leaks and undefined behavior, but like any technique, it must be verified through careful testing.

In large systems, subtle issues — such as unexpected lifetimes, hidden ownership, or misuse of non-owning pointers — can still creep in.

This section covers how to **test**, **debug**, and **visualize** RAII-based resource management.
 
## Testing and verifying RAII classes

When testing RAII wrappers, the goal is to verify:

1. The resource is properly **acquired** during construction.
2. The resource is properly **released** during destruction.
3. No resource leaks or double deletions occur.

Let’s start with a simple RAII class:

```cpp
#include <cassert>
#include <fstream>
#include <string>

class FileHandle {
    std::FILE* file_;
public:
    explicit FileHandle(const char* path, const char* mode)
        : file_(std::fopen(path, mode)) {}

    ~FileHandle() {
        if (file_) {
            std::fclose(file_);
            file_ = nullptr;
        }
    }

    std::FILE* get() const noexcept { return file_; }
    bool is_open() const noexcept { return file_ != nullptr; }
};

void test_FileHandle() {
    const char* filename = "test_file.txt";

    {
        FileHandle fh(filename, "w");
        assert(fh.is_open()); // file should be open inside scope
        std::fprintf(fh.get(), "Hello, RAII!\n");
    } // file automatically closed here

    // Verify the file exists and contains expected text
    std::ifstream fin(filename);
    assert(fin.is_open());

    std::string line;
    std::getline(fin, line);
    assert(line == "Hello, RAII!");

    fin.close();
    std::remove(filename);

    std::cout << "test_FileHandle passed.\n";
}
```

This test ensures:

* The file opens successfully.
* The destructor closes it automatically.
* The written data matches expectations.
 
## Verifying cleanup with logging or mock objects

For more complex RAII classes, logging might not be enough.
Instead, you can use **mock objects** or counters to ensure proper cleanup.

```cpp
struct Resource {
    static inline int alive_count = 0;
    Resource() { ++alive_count; }
    ~Resource() { --alive_count; }
};

void test_scoped_resource() {
    {
        Resource r1;
        Resource r2;
        assert(Resource::alive_count == 2);
    }
    assert(Resource::alive_count == 0);
    std::cout << "All resources cleaned up.\n";
}
```

This pattern is simple but powerful — by tracking how many objects are alive, you can detect missing destructors or ownership leaks during testing.
 
## Using sanitizers to detect leaks

Modern compilers offer **runtime sanitizers** that complement RAII testing:

### AddressSanitizer (ASan)

Detects:

* Use-after-free
* Double free
* Out-of-bounds memory access

Enable it with:

```shell
clang++ -fsanitize=address -g main.cpp
```

### LeakSanitizer (LSan)

Detects memory leaks even when destructors are skipped (e.g., due to abnormal termination):

```shell
clang++ -fsanitize=leak -g main.cpp
```

Example output from LSan:

```text
==1234==ERROR: LeakSanitizer: detected memory leaks
Direct leak of 40 byte(s) in 1 object at 0x...
```

RAII should prevent such leaks; if you see this message, it’s a sign your destructors aren’t firing — possibly due to static objects, cycles, or missed cleanup.
 
## Visualizing lifetime and destruction order

To debug ownership chains and destruction timing, you can add logging to constructors and destructors:

```cpp
struct Tracked {
    std::string name;
    Tracked(std::string n) : name(std::move(n)) { std::cout << "Construct " << name << "\n"; }
    ~Tracked() { std::cout << "Destruct " << name << "\n"; }
};

int main() {
    Tracked a("A");
    {
        Tracked b("B");
        Tracked c("C");
    }
    std::cout << "End of main\n";
}
```

Output:

```text
Construct A
Construct B
Construct C
Destruct C
Destruct B
End of main
Destruct A
```

You can clearly see how destructors follow reverse construction order — a key guarantee that makes RAII predictable and safe.
 
## Detecting ownership bugs with smart pointers

RAII built with smart pointers is safer, but it’s still possible to introduce *hidden ownership bugs*, such as cyclic references with `std::shared_ptr`.

### Example:

```cpp
#include <cassert>
#include <memory>
#include <iostream>

struct Resource {
    bool released = false;
    ~Resource() { released = true; }
};

void test_OwnershipBugs() {
    std::shared_ptr<Resource> res1 = std::make_shared<Resource>();
    assert(res1.use_count() == 1);

    {
        std::shared_ptr<Resource> res2 = res1;
        assert(res1.use_count() == 2);
        assert(res2.use_count() == 2);
    } // res2 destroyed, use_count decreases

    assert(res1.use_count() == 1);
    assert(!res1->released);

    res1.reset();
    // After reset, resource should be released
    // We can’t check 'released' since object is gone,
    // but no crash / double delete proves ownership is correct.

    std::cout << "test_OwnershipBugs passed.\n";
}
```
Key takeaways:

* Smart pointers manage ownership automatically.
* No manual delete needed.
* Prevents memory leaks and dangling references.
 
## Tools for debugging resource lifetime

| Tool                                        | Purpose                                        | Notes                         |
| ------------------------------------------- | ---------------------------------------------- | ----------------------------- |
| **AddressSanitizer (ASan)**                 | Detects memory corruption                      | Use with `-fsanitize=address` |
| **LeakSanitizer (LSan)**                    | Detects leaks                                  | Use with `-fsanitize=leak`    |
| **Valgrind**                                | Tracks allocations, leaks, and unfreed memory  | Slower but thorough           |
| **Dr. Memory**                              | Similar to Valgrind for Windows                | Useful for C++ desktop apps   |
| **Visual Studio Diagnostics Tools**         | Monitors heap usage, allocations, leaks        | Built-in on Windows           |
| **Static analyzers (clang-tidy, cppcheck)** | Detect missing destructors or unsafe ownership | Fast, no runtime overhead     |

Each tool complements RAII by confirming that destructors and ownership patterns work as expected.
 
## Testing RAII in multithreaded systems

When multiple threads share ownership or manage scoped locks, test for race conditions and predictable cleanup.

```cpp
#include <thread>
#include <atomic>
#include <cassert>
#include <chrono>
#include <iostream>

struct Worker {
    std::thread th;
    std::atomic<bool> done{false};

    Worker() {
        th = std::thread([this] {
            std::this_thread::sleep_for(std::chrono::milliseconds(100));
            done = true;
        });
    }

    ~Worker() {
        if (th.joinable()) th.join();
    }
};

void test_Worker() {
    Worker w;
    // Should start as not done
    assert(!w.done);

    // Wait a bit longer than the worker’s job
    std::this_thread::sleep_for(std::chrono::milliseconds(150));
    assert(w.done); // should be done by now

    // Test automatic joining (no exception, no joinable thread left)
    {
        Worker temp;
        std::this_thread::sleep_for(std::chrono::milliseconds(150));
        assert(temp.done);
    } // destructor automatically joins thread

    // Another sanity check: if Worker goes out of scope early, destructor still joins safely
    auto start = std::chrono::steady_clock::now();
    {
        Worker shortJob;
    }
    auto end = std::chrono::steady_clock::now();
    auto duration = std::chrono::duration_cast<std::chrono::milliseconds>(end - start).count();
    assert(duration >= 100 && "Destructor should block until thread finishes");

    std::cout << "test_Worker passed.\n";
}

```
The test verifies:

* The background job runs and completes.
* The destructor waits (joins) before returning.
* No dangling threads or race conditions occur.

## Common mistakes to watch for

RAII is conceptually simple — acquire a resource in a constructor, release it in a destructor —
but many subtle mistakes break its guarantees. Let’s go through the most common ones with examples.
 
### Forgetting to delete the copy constructor

**Problem:**

A resource-owning class that can be copied will cause double-deletes.

```cpp
struct BadFile {
    FILE* f;
    BadFile(const char* path) { f = fopen(path, "w"); }
    ~BadFile() { fclose(f); }  //   both copies will call fclose
};

void example() {
    BadFile a("out.txt");
    BadFile b = a; // shallow copy — disaster!
}
```

**Fix:**

Delete copy operations and implement move semantics.

```cpp
struct SafeFile {
    FILE* f;
    SafeFile(const char* path) { f = fopen(path, "w"); }
    ~SafeFile() { if (f) fclose(f); }

    SafeFile(const SafeFile&) = delete;
    SafeFile& operator=(const SafeFile&) = delete;

    SafeFile(SafeFile&& other) noexcept : f(other.f) { other.f = nullptr; }
    SafeFile& operator=(SafeFile&& other) noexcept {
        if (this != &other) {
            if (f) fclose(f);
            f = other.f;
            other.f = nullptr;
        }
        return *this;
    }
};
```
 
### Forgetting to mark destructor `noexcept`

If a destructor throws during stack unwinding, `std::terminate()` is called.

```cpp
struct BadRAII {
    ~BadRAII() { throw std::runtime_error("oops"); } //   unsafe
};
```

**Fix:**

Always mark destructors `noexcept` (which is the default unless overridden).

```cpp
struct GoodRAII {
    ~GoodRAII() noexcept {
        try { /* cleanup */ } catch (...) { /* log but don't rethrow */ }
    }
};
```
 
### Relying on destructor timing in multi-threaded code

Destructors may not run when you expect — especially if you pass smart pointers between threads.

```cpp
void example() {
    auto ptr = std::make_shared<int>(42);
    std::thread t([ptr] { std::this_thread::sleep_for(std::chrono::seconds(1)); });
    t.detach(); //   thread may still use ptr after main exits
}
```

**Fix:**

Join threads or use RAII-managed thread wrappers (`std::jthread` in C++20).
 
### Mixing manual `new`/`delete` with smart pointers

**Problem:**

Double-deletion or leak if ownership isn’t clear.

```cpp
std::shared_ptr<int> p(new int(5));
int* raw = p.get();
delete raw; //   UB: shared_ptr will delete it again
```

**Fix:**

Never manually delete memory owned by a smart pointer.
 
### Not using RAII for every resource

RAII isn’t just for heap memory. Forgetting it for other resources causes subtle bugs.

| Resource Type | Wrong                          | Correct                       |
| ------------- | ------------------------------ | ----------------------------- |
| Mutex         | `lock()` / `unlock()` manually | `std::lock_guard<std::mutex>` |
| File          | `fopen` / `fclose` manually    | `std::fstream` or custom RAII |
| Memory        | `malloc` / `free`              | `std::unique_ptr`             |
| Socket        | `open` / `close`               | Custom RAII socket wrapper    |
| Thread        | `start` / `join`               | `std::jthread`                |
 
### Returning references to destroyed objects

A common ownership logic mistake:

```cpp
const std::string& badFunc() {
    std::string temp = "RAII fail";
    return temp; //   dangling reference
}
```
 
### Forgetting to join threads (classic example)

```cpp
void badThreadExample() {
    std::thread t([] { /* work */ });
    // forgot t.join() or t.detach()
} //   std::terminate() called
```

**Fix:** Use RAII to enforce joining automatically.

```cpp
struct ThreadJoiner {
    std::thread& t;
    ~ThreadJoiner() { if (t.joinable()) t.join(); }
};
```
 
### Mixing RAII and manual lifecycle calls

If you add a `.close()` or `.release()` method, make sure it’s safe to call multiple times.

```cpp
struct File {
    FILE* f{};
    void close() { if (f) { fclose(f); f = nullptr; } }
    ~File() { close(); } // safe double call
};
```
 
### Forgetting exception safety

Constructors that throw before acquiring all resources can leak.

```cpp
struct Multi {
    std::unique_ptr<int> a;
    FILE* f;

    Multi() : a(std::make_unique<int>(42)), f(fopen("file.txt", "w")) {
        if (!f) throw std::runtime_error("cannot open file");
    }
    ~Multi() { if (f) fclose(f); } // no leak since unique_ptr cleans up
};
```
 
 
## Summary

Testing and debugging RAII isn’t just about finding leaks — it’s about confirming that **ownership and lifetime semantics behave exactly as intended**.

RAII’s determinism makes it far easier to reason about cleanup, but only if:

* You verify destructor behavior explicitly.
* You test exceptional and multithreaded paths.
* You use sanitizers and analyzers regularly.

When done right, your codebase develops an invaluable property:
**resources always clean up themselves — even in chaos.**
 