# RAII Quick Checklist: 10 Rules Before You Ship

1. **No raw `new` or `delete` anywhere.**
   If you see them, refactor to `std::unique_ptr`, `std::make_shared`, or a proper RAII wrapper.

2. **Every resource has a clear owner.**
   If multiple objects reference the same thing, define *who owns* and *who borrows* it.

3. **Destruction equals cleanup.**
   Verify your destructors actually release the resource they acquired.

4. **No manual cleanup in `try/catch` or `goto` blocks.**
   Let destructors handle rollback automatically.

5. **Use move semantics, not copy semantics, for ownership.**
   Make owning types non-copyable but movable when transfer is needed.

6. **Watch lifetime of references and raw pointers.**
   If an object outlives its owner, use `std::weak_ptr` or rethink your design.

7. **Always prefer `std::lock_guard` or `std::scoped_lock` over manual `lock()`/`unlock()`.**
   Mutex leaks are just as bad as memory leaks.

8. **Validate resource release in tests.**
   Add asserts, logs, or counters to confirm destructors fire when expected.

9. **Design RAII types for one purpose only.**
   A `ScopedTimer` measures time. A `ScopedTransaction` manages rollbacks. Keep them small and composable.

10. **Think in scopes, not statements.**
    Every `{}` block is an opportunity for deterministic cleanup.

## Bonus Habit:

Before adding a new resource (file, socket, thread, lock, GPU buffer), ask:

> “Can I wrap this in a small RAII object that guarantees it’s cleaned up automatically?”

If the answer is yes — do it.
That’s how RAII codebases stay safe, elegant, and easy to reason about, no matter how large they grow.
