# RAII Debugging Checklist

Even well-written RAII code can hide subtle bugs — leaks, double frees, or races — especially when destructors depend on complex ownership chains.

Here’s a quick checklist and tool guide to help you verify your RAII implementations.
 
## Memory leaks and dangling pointers

**Symptoms:**

* Destructor not called when expected.
* Memory usage grows unbounded.
* Valgrind reports “definitely lost” memory.

**Check:**

```bash
valgrind --leak-check=full ./your_program
```

**Common Causes:**

* Shared ownership cycles (`std::shared_ptr` loops).
* Missing `delete` in custom RAII destructors.
* Non-virtual base destructors in polymorphic classes.

**Fix Tips:**

* Use `std::weak_ptr` to break cycles.
* Verify destructors with `std::cout` or breakpoints.
* Ensure base classes with virtual functions have `virtual ~Base()`.
 
## Resource lifetime and order of destruction

**Symptoms:**

* Crash during shutdown.
* Resources used after being released.

**Check:**

```bash
ASAN_OPTIONS=verbosity=1:halt_on_error=1 ./your_program
```

**With AddressSanitizer (ASan):**

```bash
clang++ -fsanitize=address -g main.cpp -o main
./main
```

**Common Causes:**

* Static/global objects depending on each other’s destruction order.
* Referencing other objects during destructor calls.

**Fix Tips:**

* Avoid globals; prefer function-local statics (Meyers’ singleton).
* Limit cross-object cleanup dependencies.
* Use logging in destructors to trace destruction order.
 
## Threading and synchronization errors

**Symptoms:**

* Random crashes or deadlocks.
* Destructors not called because threads never terminate.
* Use-after-free when passing shared_ptr between threads.

**Check:**

```bash
clang++ -fsanitize=thread -g main.cpp -o main
./main
```

**ThreadSanitizer (TSan) detects:**

* Data races.
* Double locking or unjoined threads.
* Access after destruction.

**Fix Tips:**

* Use `std::jthread` (C++20) for automatic joining.
* Use `std::lock_guard` or `std::scoped_lock`.
* Keep ownership simple: prefer `unique_ptr` over `shared_ptr`.
 
## Exception safety and cleanup validation

**Symptoms:**

* Leaks or partial cleanup when constructors throw.
* Double-release if exceptions occur before full initialization.

**Check:**
Run with ASan/Valgrind under an artificially thrown exception in constructors.

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

If your destructor or RAII logic is solid, this should **not leak**.
 
## Logging and assertions for manual verification

In debug builds, always verify that resources are acquired and released symmetrically.

```cpp
struct DebugRAII {
    static inline int counter = 0;
    DebugRAII() { ++counter; std::cout << "Acquire #" << counter << "\n"; }
    ~DebugRAII() { std::cout << "Release #" << counter-- << "\n"; }
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
 
## Quick summary table

| Tool                 | Detects                           | Command Example                         |
| -------------------- | --------------------------------- | --------------------------------------- |
| **Valgrind**         | Memory leaks, double free         | `valgrind --leak-check=full ./a.out`    |
| **ASan**             | Buffer overflow, use-after-free   | `clang++ -fsanitize=address main.cpp`   |
| **TSan**             | Data races, unjoined threads      | `clang++ -fsanitize=thread main.cpp`    |
| **UBSan**            | Undefined behavior                | `clang++ -fsanitize=undefined main.cpp` |
| **Static analyzers** | Lifetime, leaks, copy/move errors | `clang-tidy main.cpp`                   |

