# RAII Verification and Definitive Design Rules

## Testing and verifying RAII classes

When testing RAII wrappers, the goal is to verify that resources are properly 
acquired during construction and properly released during destruction. Let's 
test the `FileHandle` class we built in Chapter *RAII In Practice: Patterns And Examples* to demonstrate these verification 
techniques.

### Example: Testing a FileHandle class

```cpp
void test_FileHandle() {
    const char* filename = "test_file.txt";
    
    // Test 1: Basic acquisition and release
    {
        FileHandle fh(filename, "w");
        assert(fh.is_open());
        std::fprintf(fh.get(), "Hello, RAII!\n");
    } // File should be closed here
    
    // Test 2: Verify the file was actually written and closed
    std::ifstream verify(filename);
    assert(verify.is_open());
    std::string line;
    std::getline(verify, line);
    assert(line == "Hello, RAII!");
    verify.close();
    
    // Test 3: Move operations
    {
        FileHandle original(filename, "w");
        FileHandle moved = std::move(original);
        
        assert(!original.is_open());  // Source is now empty
        assert(moved.is_open());       // Destination owns the file
        
        std::fprintf(moved.get(), "Moved successfully\n");
    }
    
    // Cleanup
    std::remove(filename);
    std::cout << "All FileHandle tests passed.\n";
}
```

This test ensures:
- The file opens successfully.
- The destructor closes it automatically.
- The written data matches expectations.

## Verifying cleanup with logging or mock objects

For more complex RAII classes, logging might not be enough. Instead, you can use mock objects or counters to ensure proper cleanup.

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

## RAII debugging checklist

Even well-written RAII code can hide subtle bugs. Here's a systematic approach to debugging common issues:

### Memory leaks and dangling pointers

**Symptoms:**
- Destructor not called when expected.
- Memory usage grows unbounded.
- Valgrind reports "definitely lost" memory.

**Check with Valgrind:**
```bash
valgrind --leak-check=full ./your_program
```

**Common Causes:**
- Shared ownership cycles (`std::shared_ptr` loops).
- Missing delete in custom RAII destructors.
- Non-virtual base destructors in polymorphic classes.

**Fix Tips:**
- Use `std::weak_ptr` to break cycles.
- Verify destructors with `std::cout` or breakpoints.
- Ensure base classes with virtual functions have `virtual ~Base()`.

### Resource lifetime and order of destruction

**Symptoms:**
- Crash during shutdown.
- Resources used after being released.

**Check with AddressSanitizer (ASan):**
```bash
clang++ -fsanitize=address -g main.cpp -o main
./main
```

Or with environment variables:
```bash
ASAN_OPTIONS=verbosity=1:halt_on_error=1 ./your_program
```

**Common Causes:**
- Static/global objects depending on each other's destruction order.
- Referencing other objects during destructor calls.

**Fix Tips:**
- Avoid globals; prefer function-local statics (Meyers' singleton).
- Limit cross-object cleanup dependencies.
- Use logging in destructors to trace destruction order.

### Threading and synchronization errors

**Symptoms:**
- Random crashes or deadlocks.
- Destructors not called because threads never terminate.
- Use-after-free when passing `shared_ptr` between threads.

**Check with ThreadSanitizer (TSan):**
```bash
clang++ -fsanitize=thread -g main.cpp -o main
./main
```

ThreadSanitizer detects:
- Data races.
- Double locking or unjoined threads.
- Access after destruction.

**Fix Tips:**
- Use `std::jthread` (C++20) for automatic joining.
- Use `std::lock_guard` or `std::scoped_lock`.
- Keep ownership simple: prefer `unique_ptr` over `shared_ptr`.

### Exception safety and cleanup validation

**Symptoms:**
- Leaks or partial cleanup when constructors throw.
- Double-release if exceptions occur before full initialization.

**Check:** Run with ASan/Valgrind under an artificially thrown exception in constructors.

Example:
```cpp
struct Test {
    std::unique_ptr<int> p;
    
    Test() {
        p = std::make_unique<int>(42);
        throw std::runtime_error("test exception");
    }
};
```

If your destructor or RAII logic is solid, this should not leak.

## Using sanitizers to detect leaks

Modern compilers offer runtime sanitizers that complement RAII testing:

### AddressSanitizer (ASan)

Detects:
- Use-after-free
- Double free
- Out-of-bounds memory access

Enable it with:
```bash
clang++ -fsanitize=address -g main.cpp
```

### LeakSanitizer (LSan)

Detects memory leaks even when destructors are skipped (e.g., due to abnormal termination):

```bash
clang++ -fsanitize=leak -g main.cpp
```

Example output from LSan:
```
==1234==ERROR: LeakSanitizer: detected memory leaks
Direct leak of 40 byte(s) in 1 object at 0x...
```

RAII should prevent such leaks; if you see this message, it's a sign your destructors aren't firing — possibly due to static objects, cycles, or missed cleanup.

### UndefinedBehaviorSanitizer (UBSan)

```bash
clang++ -fsanitize=undefined -g main.cpp
```

## Tools for debugging resource lifetime

| Tool | Purpose | Notes |
|------|---------|-------|
| AddressSanitizer (ASan) | Detects memory corruption | Use with `-fsanitize=address` |
| LeakSanitizer (LSan) | Detects leaks | Use with `-fsanitize=leak` |
| ThreadSanitizer (TSan) | Detects data races, unjoined threads | Use with `-fsanitize=thread` |
| UBSan | Detects undefined behavior | Use with `-fsanitize=undefined` |
| Valgrind | Tracks allocations, leaks, and unfreed memory | Slower but thorough |
| Dr. Memory | Similar to Valgrind for Windows | Useful for C++ desktop apps |
| Visual Studio Diagnostics Tools | Monitors heap usage, allocations, leaks | Built-in on Windows |
| Static analyzers (clang-tidy, cppcheck) | Detect missing destructors or unsafe ownership | Fast, no runtime overhead |

Each tool complements RAII by confirming that destructors and ownership patterns work as expected.

## Visualizing lifetime and destruction order

To debug ownership chains and destruction timing, you can add logging to constructors and destructors:

```cpp
struct Tracked {
    std::string name;
    
    Tracked(std::string n) : name(std::move(n)) {
        std::cout << "Construct " << name << "\n";
    }
    
    ~Tracked() {
        std::cout << "Destruct " << name << "\n";
    }
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
```
Construct A
Construct B
Construct C
Destruct C
Destruct B
End of main
Destruct A
```

You can clearly see how destructors follow reverse construction order — a key guarantee that makes RAII predictable and safe.

## Logging and assertions for manual verification

In debug builds, always verify that resources are acquired and released symmetrically.

```cpp
struct DebugRAII {
    static inline int counter = 0;
    
    DebugRAII() {
        ++counter;
        std::cout << "Acquire #" << counter << "\n";
    }
    
    ~DebugRAII() {
        std::cout << "Release #" << counter-- << "\n";
    }
};
```

Run with assertions:

```cpp
{
    DebugRAII d1, d2;
    assert(DebugRAII::counter == 2);
}
assert(DebugRAII::counter == 0);
```

## Common mistakes to watch for

RAII is conceptually simple — acquire a resource in a constructor, release it in a destructor — but many subtle mistakes break its guarantees.

### Forgetting to delete the copy constructor

**Problem:** A resource-owning class that can be copied will cause double-deletes.

```cpp
struct BadFile {
    FILE* f;
    BadFile(const char* path) { f = fopen(path, "w"); }
    ~BadFile() { fclose(f); } // both copies will call fclose
};

void example() {
    BadFile a("out.txt");
    BadFile b = a; // shallow copy — disaster!
}
```

**Fix:** Delete copy operations and implement move semantics.

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
    ~BadRAII() { throw std::runtime_error("oops"); } // unsafe
};
```

**Fix:** Always mark destructors `noexcept` (which is the default unless overridden).

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
    t.detach(); // thread may still use ptr after main exits
}
```

**Fix:** Join threads or use RAII-managed thread wrappers (`std::jthread` in C++20).

### Mixing manual `new`/`delete` with smart pointers

**Problem:** Double-deletion or leak if ownership isn't clear.

```cpp
std::shared_ptr<int> p(new int(5));
int* raw = p.get();
delete raw; // UB: shared_ptr will delete it again
```

**Fix:** Never manually delete memory owned by a smart pointer.

### Not using RAII for every resource

RAII isn't just for heap memory. Forgetting it for other resources causes subtle bugs.

| Resource Type | Wrong | Correct |
|---|---|---|
| Mutex | `lock()` / `unlock()` manually | `std::lock_guard<std::mutex>` |
| File | `fopen` / `fclose` manually | `std::fstream` or custom RAII |
| Memory | `malloc` / `free` | `std::unique_ptr` |
| Socket | `open` / `close` | Custom RAII socket wrapper |
| Thread | `start` / `join` | `std::jthread` |

### Returning references to destroyed objects

A common ownership logic mistake:

```cpp
const std::string& badFunc() {
    std::string temp = "RAII fail";
    return temp; // dangling reference
}
```

### Forgetting to join threads

```cpp
void badThreadExample() {
    std::thread t([] { /* work */ });
    // forgot t.join() or t.detach()
} // std::terminate() called
```

**Fix:** Use RAII to enforce joining automatically.

```cpp
struct ThreadJoiner {
    std::thread& t;
    ~ThreadJoiner() { if (t.joinable()) t.join(); }
};
```

### Mixing RAII and manual lifecycle calls

If you add a `.close()` or `.release()` method, make sure it's safe to call multiple times.

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
class Multi {
    std::unique_ptr<int> a;
    FILE* f;
    
public:
    Multi() : a(std::make_unique<int>(42)), f(fopen("file.txt", "w")) {
        if (!f) throw std::runtime_error("cannot open file");
    }
    
    ~Multi() { if (f) fclose(f); } // no leak since unique_ptr cleans up
};
```

