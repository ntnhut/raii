# RAII Design Checklist and Best Practices


## RAII design checklist
Designing effective RAII classes means following a few key principles consistently.

### Constructor = Acquire, Destructor = Release

**Rule:** Each RAII class should ensure resource acquisition happens during construction, and cleanup happens during destruction — no exceptions, no manual cleanup elsewhere.

Example:

```cpp
#include <cstdio>

class FileHandle {
    FILE* file_;
    
public:
    explicit FileHandle(const char* path, const char* mode)
        : file_(std::fopen(path, mode)) {
        if (!file_)
            throw std::runtime_error("Failed to open file");
    }
    
    ~FileHandle() noexcept {
        if (file_) std::fclose(file_);
    }
};
```

If the constructor throws, no valid object is created, and therefore the destructor is never called — meaning no risk of double cleanup.

### Use the Rule of Five (or Zero)

Whenever you manage a raw resource, define or delete the special member functions appropriately:

| Function | Common Rule |
|----------|-------------|
| Copy constructor | Usually `= delete` |
| Copy assignment | Usually `= delete` |
| Move constructor | Implement for ownership transfer |
| Move assignment | Implement for ownership transfer |
| Destructor | Always `noexcept` |

Example:

```cpp
class Socket {
    int fd_;
    
public:
    explicit Socket(int fd) : fd_(fd) {}
    
    ~Socket() { if (fd_ != -1) close(fd_); }
    
    Socket(const Socket&) = delete;
    Socket& operator=(const Socket&) = delete;
    
    Socket(Socket&& other) noexcept : fd_(other.fd_) {
        other.fd_ = -1;
    }
    
    Socket& operator=(Socket&& other) noexcept {
        if (this != &other) {
            if (fd_ != -1) close(fd_);
            fd_ = other.fd_;
            other.fd_ = -1;
        }
        return *this;
    }
};
```

This ensures ownership can be transferred safely without leaks or double releases.

### Prefer composition over inheritance

Inheritance often introduces fragile ownership and lifetime dependencies between base and derived classes. RAII works best when each object directly owns its resources — that's composition.

**Bad (inheritance-based ownership):**

```cpp
struct BaseResource {
    BaseResource() { std::cout << "Base acquired\n"; }
    virtual ~BaseResource() { std::cout << "Base released\n"; }
};

struct Derived : BaseResource {
    Derived() { std::cout << "Derived acquired\n"; }
    ~Derived() { std::cout << "Derived released\n"; }
};
```

Here, destruction order can be confusing when multiple base classes are involved, especially with virtual destructors.

**Better (composition-based):**

```cpp
struct ScopedTimer {
    std::chrono::steady_clock::time_point start;
    
    ScopedTimer() : start(std::chrono::steady_clock::now()) {}
    
    ~ScopedTimer() {
        auto ms = std::chrono::duration_cast<std::chrono::milliseconds>(
            std::chrono::steady_clock::now() - start);
        std::cout << "Elapsed: " << ms.count() << " ms\n";
    }
};

struct Worker {
    ScopedTimer timer; // composition
    
    void run() {
        std::this_thread::sleep_for(std::chrono::milliseconds(200));
    }
};
```

Here, `Worker` composes a `ScopedTimer` — no inheritance, no virtual destructors, and cleanup happens automatically when `Worker` is destroyed. That's the essence of RAII: each class manages its own resources independently.

### Use smart pointers correctly (including arrays)

Smart pointers embody RAII for dynamic memory. They ensure deterministic destruction and eliminate the need for `delete` or `delete[]`.

| Resource Type | Recommended Smart Pointer |
|---|---|
| Single object | `std::unique_ptr<T>` |
| Dynamic array | `std::unique_ptr<T[]>` |
| Shared ownership | `std::shared_ptr<T>` |
| Observing only | `std::weak_ptr<T>` or raw pointer |

Example: Managing a dynamic array

```cpp
void test_unique_ptr_array() {
    std::unique_ptr<int[]> arr = std::make_unique<int[]>(5);
    for (int i = 0; i < 5; ++i)
        arr[i] = i * 10;
    for (int i = 0; i < 5; ++i)
        std::cout << arr[i] << " ";
    std::cout << "\n";
} // arr automatically deleted here
```

Unlike `std::unique_ptr<T>`, the array version automatically calls `delete[]` (not `delete`), ensuring safe and correct cleanup of all elements.

### Handle exceptions gracefully

Constructors may throw exceptions if resource acquisition fails. When this happens, destructors of already-constructed subobjects still run — which means partially constructed objects clean up safely.

Example:

```cpp
class ConfigLoader {
    std::ifstream file_;
    
public:
    ConfigLoader(const std::string& path)
        : file_(path) {
        if (!file_)
            throw std::runtime_error("Cannot open config file: " + path);
    }
    
    void parse() {
        std::string line;
        while (std::getline(file_, line)) {
            // process line
        }
    }
};
```

What happens here:
- The constructor tries to open the file.
- If it fails, it throws immediately.
- Since the object isn't fully constructed, its destructor is not called — but that's safe because no resource was acquired successfully.
- If it succeeds, then `file_`'s destructor (from `std::ifstream`) automatically closes the file later, even if an exception happens during parsing.

This demonstrates one of RAII's greatest strengths: automatic exception safety — you don't need try/catch just to close resources.

### Make destructors simple and deterministic

Keep destructors minimal:
- Don't throw exceptions.
- Avoid blocking operations (especially in multithreaded contexts).
- Log or fail silently instead of propagating errors.

```cpp
~Socket() noexcept {
    if (fd_ != -1 && close(fd_) == -1) {
        std::cerr << "Warning: socket close failed\n";
    }
}
```

### Be thread-safe when needed

RAII helps avoid deadlocks and leaks in multithreaded code.

Example using scoped locks:

```cpp
std::mutex m;

void safe_increment(int& counter) {
    std::lock_guard<std::mutex> guard(m);
    ++counter;
} // guard releases automatically
```

### Avoid hidden ownership

Ownership should always be explicit. If your RAII wrapper borrows a resource, make that clear by:
- Naming conventions (`BorrowedX`, `View`, `Ref`), or
- Documentation comments (`// does not own`).

### Test for symmetry

Each acquisition should have one matching release. Use assertions or counters to ensure deterministic cleanup.

```cpp
struct Tracker {
    static inline int active = 0;
    
    Tracker() { ++active; }
    ~Tracker() { --active; }
};

void test_tracker() {
    {
        Tracker a, b;
        assert(Tracker::active == 2);
    }
    assert(Tracker::active == 0);
}
```

### Layer RAII for higher-level safety

You can combine multiple RAII objects to manage complex systems:

```cpp
struct DatabaseTransaction {
    bool active = true;
    
    ~DatabaseTransaction() {
        if (active) std::cout << "Rollback\n";
    }
    
    void commit() {
        active = false;
        std::cout << "Commit\n";
    }
};

void test_transaction() {
    DatabaseTransaction txn;
    try {
        // some work
        throw std::runtime_error("error");
        txn.commit();
    } catch (...) {
        std::cout << "Exception caught\n";
    }
}
```

Output:
```
Exception caught
Rollback
```

Even with early returns or exceptions, RAII ensures rollback happens automatically — no forgotten cleanup.

## Essential RAII best practices for modern C++

Resource Acquisition Is Initialization (**RAII**) is the cornerstone of robust C++ resource management. These best practices ensure reliable, exception-safe code by tying a resource's lifecycle to an object's lifetime.

### Core principles of RAII

* **Encapsulate Every Resource:** Wrap **every** resource (memory, files, network handles, locks, buffers, threads) inside a dedicated class that owns it.
* **Destruction Equals Cleanup:** The resource must be acquired in the constructor and released in the destructor. Verify that your destructors actually release the resource they acquired.
* **Think in Lifetimes and Scopes:** Design systems by clearly defining who creates a resource, who destroys it, and when it dies. Every `{}` block is an opportunity for deterministic cleanup.


### Memory management and ownership

* **Avoid Raw `new` and `delete`:** They should **never** appear in production application code. Refactor to use proper RAII wrappers.
* **Use Smart Pointers:** Prefer `std::unique_ptr` for exclusive ownership. Use `std::shared_ptr` only when genuinely needing shared ownership, often via `std::make_shared`.
* **Clear Ownership:** Every resource must have a clear owner. If multiple objects reference the same resource, define who **owns** and who **borrows** it.
* **Ownership Transfer via Move Semantics:** Use `std::move` to explicitly transfer ownership. Make owning types non-copyable but movable for clarity and compiler enforcement.


### Safety and exception handling

* **Handle Exceptions Safely:** Let destructors perform cleanup automatically. **Never** depend on manual cleanup calls in `try`/`catch` blocks or use `goto` for rollback.
* **Mutex Safety:** Always prefer **`std::lock_guard`** or **`std::scoped_lock`** over manual `lock()`/`unlock()`. Mutex leaks are as severe as memory leaks.
* **Watch Non-Owning References:** Be extremely careful with the lifetime of raw pointers and references. If a non-owning object might outlive its owner, use **`std::weak_ptr`** or rethink the design to prevent dangling references.


### Design and testing

* **Keep RAII Types Small and Single-Purpose:** Design RAII classes for **one responsibility only**. A `ScopedLock` manages a mutex; a `ScopedTimer` measures time. Prefer composition over inheritance.
* **Validate Resource Release in Tests:** Add asserts, logs, or counters in your tests to confirm that destructors fire and cleanup happens exactly when expected, especially in complex compositions.

## Bonus habit

Before adding a new resource (file, socket, thread, lock, GPU buffer), ask:

> "Can I wrap this in a small RAII object that guarantees it's cleaned up automatically?"

If the answer is yes—do it. That's how RAII codebases stay safe, elegant, and easy to reason about, no matter how large they grow.

## The RAII mindset

Mastering RAII changes how you think about code.

Instead of fighting memory and resource management, you start designing flows of ownership. Every scope becomes a safe boundary. Every destructor, a quiet guardian.

This philosophy scales—from a small scoped lock to a high-performance system managing thousands of threads and transactions. When every object owns what it needs, and releases what it owns, your program behaves predictably under stress, load, and failure.

## Summary

Testing and debugging RAII isn't just about finding leaks — it's about confirming that ownership and lifetime semantics behave exactly as intended.

RAII's determinism makes it far easier to reason about cleanup, but only if:

- You verify destructor behavior explicitly.
- You test exceptional and multithreaded paths.
- You use sanitizers and analyzers regularly.
- You watch for common pitfalls like copy operations and destructor exceptions.
- You follow the 10 definitive rules consistently.
- You design RAII types with clear, single responsibility.

When done right, your codebase develops an invaluable property: **resources always clean up themselves — even in chaos.**

RAII isn't just a technique—it's a way to reason about program correctness. Master it, and your C++ code becomes safer, cleaner, and more maintainable at every scale.