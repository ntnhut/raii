# RAII Design Checklist

Designing effective RAII classes means following a few key principles consistently. 
 
## Constructor = Acquire, Destructor = Release

**Rule:**
Each RAII class should ensure resource acquisition happens during construction, and cleanup happens during destruction — no exceptions, no manual cleanup elsewhere.

**Example:**

```cpp
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
 
## Use the Rule of Five (or Zero)

Whenever you manage a raw resource, define or delete the special member functions appropriately:

| Function         | Common Rule                      |
| ---------------- | -------------------------------- |
| Copy constructor | Usually `= delete`               |
| Copy assignment  | Usually `= delete`               |
| Move constructor | Implement for ownership transfer |
| Move assignment  | Implement for ownership transfer |
| Destructor       | Always `noexcept`                |

**Example:**

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

This ensures ownership can be *transferred* safely without leaks or double releases.
 
## Prefer composition over inheritance

Inheritance often introduces fragile ownership and lifetime dependencies between base and derived classes.
RAII works best when *each object directly owns its resources* — that’s **composition**.

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

Here, `Worker` *composes* a `ScopedTimer` — no inheritance, no virtual destructors, and cleanup happens automatically when `Worker` is destroyed.

That’s the essence of RAII: **each class manages its own resources independently**.
 
## Use smart pointers correctly (including arrays)

Smart pointers embody RAII for dynamic memory.

They ensure deterministic destruction and eliminate the need for `delete` or `delete[]`.

| Resource Type    | Recommended Smart Pointer         |
| ---------------- | --------------------------------- |
| Single object    | `std::unique_ptr<T>`              |
| Dynamic array    | `std::unique_ptr<T[]>`            |
| Shared ownership | `std::shared_ptr<T>`              |
| Observing only   | `std::weak_ptr<T>` or raw pointer |

**Example: Managing a dynamic array**

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
 
## Handle exceptions gracefully

Constructors may throw exceptions if resource acquisition fails.

When this happens, destructors of already-constructed subobjects still run — which means partially constructed objects clean up safely.

**Example:**

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

**What happens here:**

* The constructor tries to open the file.
* If it fails, it throws immediately.
* Since the object isn’t fully constructed, its destructor is **not called** — but that’s safe because no resource was acquired successfully.
* If it succeeds, then `file_`’s destructor (from `std::ifstream`) automatically closes the file later, even if an exception happens during parsing.

This demonstrates one of RAII’s greatest strengths: **automatic exception safety** — you don’t need `try/catch` just to close resources.
 
## Make destructors simple and deterministic

Keep destructors minimal:

* Don’t throw exceptions.
* Avoid blocking operations (especially in multithreaded contexts).
* Log or fail silently instead of propagating errors.

```cpp
~Socket() noexcept {
    if (fd_ != -1 && close(fd_) == -1) {
        std::cerr << "Warning: socket close failed\n";
    }
}
```
 
## Be thread-safe when needed

RAII helps avoid deadlocks and leaks in multithreaded code.

Example using scoped locks:

```cpp
std::mutex m;
void safe_increment(int& counter) {
    std::lock_guard<std::mutex> guard(m);
    ++counter;
} // guard releases automatically
```
 
## Avoid hidden ownership

Ownership should always be explicit.

If your RAII wrapper *borrows* a resource, make that clear by:

* Naming conventions (`BorrowedX`, `View`, `Ref`), or
* Documentation comments (`// does not own`).
 
## Test for symmetry

Each acquisition should have one matching release.

Use assertions or counters to ensure deterministic cleanup.

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
 
## Layer RAII for higher-level safety

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
// Output: Exception caught
// Rollback
```

Even with early returns or exceptions, RAII ensures rollback happens automatically — no forgotten cleanup.
 
