# Project 1 Design Document — Threads

## Table of Contents

- [Recommended Implementation Order](#recommended-implementation-order)
- [Task 1 — Alarm Clock](#task-1--alarm-clock)
- [Task 2 — Priority Scheduling](#task-2--priority-scheduling)
- [Task 3 — Priority Donation](#task-3--priority-donation)
- [Task 4 — thread_set_priority / thread_get_priority](#task-4--thread_set_priority--thread_get_priority)
- [Testing Procedures](#testing-procedures)

---

## Recommended Implementation Order

1. Implement **Alarm Clock** and verify sleep-related tests.
2. Implement **basic priority scheduling** (ready-list ordering + preemption).
3. Update **synchronization primitives** to wake highest-priority waiters.
4. Implement **priority donation** and verify inversion-related tests.

> Each later task builds on the previous one. Do **not** skip ahead.

---

## Task 1 — Alarm Clock

**Goal:** Replace the busy-wait loop in `timer_sleep()` with a blocking sleep mechanism.

### Files to Modify


| File                   | What to Do                                                                                                                                                                               |
| ---------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `src/threads/thread.h` | Add a `wakeup_tick` field (type `int64_t`) to `struct thread`.                                                                                                                           |
| `src/threads/thread.c` | Initialize `wakeup_tick` in `init_thread()`. Declare and initialize a global `sleep_list` (`struct list`). Provide a `list_less_func` comparator that orders by `wakeup_tick` ascending. |
| `src/devices/timer.c`  | Rewrite `timer_sleep()` to block instead of spin. Modify `timer_interrupt()` to wake threads whose deadline has passed.                                                                  |
| `src/devices/timer.h`  | No changes expected (API is unchanged).                                                                                                                                                  |


### Files to Read / Understand First


| File                      | Why                                                                                             |
| ------------------------- | ----------------------------------------------------------------------------------------------- |
| `src/lib/kernel/list.h`   | Understand `list_insert_ordered()`, `list_less_func`, `list_entry`, iteration macros.           |
| `src/lib/kernel/list.c`   | Implementation of the ordered-insert and traversal helpers.                                     |
| `src/threads/interrupt.h` | Understand `intr_disable()`, `intr_set_level()`, `intr_context()`, `enum intr_level`.           |
| `src/threads/thread.c`    | Understand `thread_block()`, `thread_unblock()`, `thread_current()`, the existing `ready_list`. |


### Step-by-Step Plan

**Step 1 — Add a wakeup timestamp to** `struct thread`

- Add `int64_t wakeup_tick;` to `struct thread` in `thread.h`.
- In `init_thread()` (`thread.c`), initialize it to `0`.

**Step 2 — Create a global sleep queue**

- In `thread.c` (or `timer.c` — either works, but `timer.c` is more cohesive):
  - Declare `static struct list sleep_list;`.
  - Initialize it in `timer_init()` (or `thread_init()`).
  - Write a comparator: `static bool wakeup_tick_less(const struct list_elem *a, const struct list_elem *b, void *aux)` that compares `wakeup_tick` of the containing `struct thread`.

**Step 3 — Rewrite** `timer_sleep()`

- Validate: if `ticks <= 0`, return immediately.
- Compute the absolute deadline: `thread_current()->wakeup_tick = timer_ticks() + ticks;`.
- Disable interrupts (`enum intr_level old_level = intr_disable();`).
- Insert the current thread into `sleep_list` in sorted order via `list_insert_ordered()`.
- Call `thread_block()` to deschedule the thread.
- Restore interrupt level (`intr_set_level(old_level);`).

**Step 4 — Update** `timer_interrupt()`

- After `ticks++; thread_tick();`, iterate `sleep_list` from the front.
- While the front element's `wakeup_tick <= ticks`, remove it and call `thread_unblock()`.
- Because the list is sorted, stop as soon as a thread's deadline is still in the future.

### Things to Consider

- **Interrupt safety:** `timer_sleep()` is called with interrupts ON. You must disable interrupts before touching `sleep_list` and before calling `thread_block()` (which asserts `INTR_OFF`).
- `timer_interrupt()` **runs in interrupt context:** You cannot call `thread_block()` or any function that sleeps. You *can* call `thread_unblock()`.
- **Edge cases:** `ticks == 0` and `ticks < 0` should return immediately without blocking (the existing code handles this via the while-loop condition — your new code must also).
- **No** `malloc`**:** Use the intrusive `struct list_elem elem` already in `struct thread` (it is free when the thread is blocked; it is not on `ready_list`).
- **Sorted insertion vs. linear scan on wake:** Inserting in order makes the interrupt handler O(k) where k = number of threads waking up this tick, instead of O(n) every tick.
- **Global tick variable:** The `ticks` variable in `timer.c` is `static` — your `timer_interrupt()` code is already inside that file, so it has access.

---

## Task 2 — Priority Scheduling

**Goal:** The scheduler must always run the highest-priority ready thread.

### Files to Modify


| File                   | What to Do                                                                                                                                                                                                                                                                                                                                       |
| ---------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `src/threads/thread.c` | Change `thread_unblock()` to insert into `ready_list` in priority order (use `list_insert_ordered()`). Change `thread_yield()` similarly. Change `next_thread_to_run()` to pop the front (highest priority). After `thread_unblock()` and `thread_create()`, check if the new thread has higher priority than the running thread — if so, yield. |
| `src/threads/thread.h` | Declare a `thread_priority_less()` comparator if you want it accessible from `synch.c`.                                                                                                                                                                                                                                                          |
| `src/threads/synch.c`  | Modify `sema_up()`: unblock the highest-priority waiter (use `list_max()` or keep waiters sorted). Modify `sema_down()`: insert into `sema->waiters` in priority order. Modify `cond_signal()`: signal the condition waiter whose semaphore has the highest-priority thread.                                                                     |
| `src/threads/synch.h`  | No struct changes expected yet.                                                                                                                                                                                                                                                                                                                  |


### Files to Read / Understand First


| File                    | Why                                                                                                                                        |
| ----------------------- | ------------------------------------------------------------------------------------------------------------------------------------------ |
| `src/threads/thread.c`  | Understand `thread_unblock()` (currently uses `list_push_back`), `thread_yield()`, `next_thread_to_run()`, `thread_create()`.              |
| `src/threads/synch.c`   | Understand `sema_down()` (pushes to back of waiters), `sema_up()` (pops from front), `cond_signal()` (pops from front of `cond->waiters`). |
| `src/lib/kernel/list.h` | `list_insert_ordered()`, `list_max()`, `list_min()`, `list_sort()`.                                                                        |


### Step-by-Step Plan

**Step 1 — Write a priority comparator**

- Write `bool thread_priority_greater(const struct list_elem *a, const struct list_elem *b, void *aux UNUSED)` that returns `true` if thread A has higher priority than thread B. (Note: `list_insert_ordered` keeps the list in ascending order, so to get highest-priority at the front, pass a "greater" comparator.)

**Step 2 — Ordered ready list**

- In `thread_unblock()`: replace `list_push_back(&ready_list, &t->elem)` with `list_insert_ordered(&ready_list, &t->elem, thread_priority_greater, NULL)`.
- In `thread_yield()`: same change for `list_push_back(&ready_list, &cur->elem)`.
- `next_thread_to_run()` already pops from the front — no change needed if the list is now sorted highest-first.

**Step 3 — Preemption on unblock / create**

- At the end of `thread_unblock()`: if `t->priority > thread_current()->priority`, set a flag or call `intr_yield_on_return()` (if in interrupt context) or `thread_yield()` (if not). Be careful: `thread_unblock()` can be called from interrupt context.
- At the end of `thread_create()`, after `thread_unblock(t)`: if the new thread's priority is higher than the current thread's, call `thread_yield()`.

**Step 4 — Update synchronization primitives**

- `sema_down()`: replace `list_push_back(&sema->waiters, ...)` with `list_insert_ordered(...)` by priority.
- `sema_up()`: instead of `list_pop_front`, use `list_max()` to find the highest-priority waiter, `list_remove()` it, then `thread_unblock()` it. After incrementing `sema->value`, yield if the unblocked thread has higher priority.
- `cond_signal()`: instead of popping the front `semaphore_elem`, find the one whose single waiter has the highest priority, remove it, and `sema_up()` it.

### Things to Consider

- **Comparator direction:** PintOS `list_insert_ordered()` inserts *before* the first element for which `less()` returns false. To keep highest-priority at the front, your comparator should return `true` when A's priority > B's priority.
- **Preemption from interrupt context:** `thread_unblock()` may be called inside `timer_interrupt()`. You cannot call `thread_yield()` from interrupt context — use `intr_yield_on_return()` instead. Check `intr_context()` to decide.
- **Re-sorting after priority changes:** When a thread's priority changes later (Task 3/4), queues may need re-sorting. Keep this in mind.
- **Idle thread:** Never yield to the idle thread. The `thread_yield()` / `next_thread_to_run()` logic already handles this since the idle thread is never on `ready_list`.

---

## Task 3 — Priority Donation

**Goal:** Prevent priority inversion when a high-priority thread waits on a lock held by a lower-priority thread.

### Files to Modify


| File                   | What to Do                                                                                                                                                                                                                                                                                                                                                                                                                                                             |
| ---------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `src/threads/thread.h` | Add to `struct thread`: `int original_priority;`, `struct list donors;` (list of threads donating to this one), `struct list_elem donor_elem;` (for being in another thread's `donors` list), `struct lock *waiting_on_lock;` (the lock this thread is blocked on, or `NULL`).                                                                                                                                                                                         |
| `src/threads/thread.c` | Initialize new fields in `init_thread()`: `original_priority = priority`, `list_init(&donors)`, `waiting_on_lock = NULL`. Add a helper `thread_recompute_priority(struct thread *t)` that sets `t->priority = max(t->original_priority, max donation from t->donors)`. Declare it in `thread.h` so `synch.c` can call it.                                                                                                                                              |
| `src/threads/synch.c`  | Modify `lock_acquire()`: before `sema_down()`, set `cur->waiting_on_lock = lock`, donate priority up the holder chain (depth limit 8). After `sema_down()`, clear `waiting_on_lock`, set `lock->holder = cur`. Modify `lock_release()`: remove all donors whose `waiting_on_lock` matches the released lock, call `thread_recompute_priority(cur)`, then set `lock->holder = NULL` and `sema_up()`. After `sema_up()`, yield if a higher-priority thread is now ready. |
| `src/threads/synch.h`  | No struct changes required (the existing `struct lock` already has a `holder` field and an embedded semaphore).                                                                                                                                                                                                                                                                                                                                                        |


### Files to Read / Understand First


| File                                            | Why                                                                                                                                             |
| ----------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------- |
| `src/threads/synch.c`                           | Understand the full `lock_acquire()` → `sema_down()` → `thread_block()` path, and the `lock_release()` → `sema_up()` → `thread_unblock()` path. |
| `src/threads/thread.h`                          | Understand the existing `struct thread` layout and the 4KB page constraint (don't bloat the struct).                                            |
| `src/tests/threads/priority-donate-one.c`       | Basic single donation: two higher-priority threads block on one lock.                                                                           |
| `src/tests/threads/priority-donate-multiple.c`  | One thread holds two locks with different-priority donors on each. Releasing one lock drops only that lock's donation.                          |
| `src/tests/threads/priority-donate-multiple2.c` | Same as above but with a different release order and an interloper thread.                                                                      |
| `src/tests/threads/priority-donate-nest.c`      | Nested donation: H donates to M, which propagates to L through a chain of two locks.                                                            |
| `src/tests/threads/priority-donate-chain.c`     | Deep nested donation chain of 7 locks and 7 donor threads (depth limit = 8).                                                                    |
| `src/tests/threads/priority-donate-lower.c`     | Calling `thread_set_priority()` to a lower value while a donation is active must not override the donation.                                     |
| `src/tests/threads/priority-donate-sema.c`      | Donation interacts correctly when the lock holder is blocked on a semaphore.                                                                    |


### Step-by-Step Plan

**Step 1 — Extend** `struct thread` **with donation-tracking fields**

In `src/threads/thread.h`, add these members to `struct thread`:

```c
int original_priority;            /* Priority assigned at creation or via thread_set_priority() */
struct list donors;               /* List of threads that have donated their priority to us */
struct list_elem donor_elem;      /* Element for being in another thread's donors list */
struct lock *waiting_on_lock;     /* The lock this thread is currently blocked on, or NULL */
```

Why each field matters:

- `original_priority` — separates the thread's "base" priority from temporary boosts. When all donations are removed the thread falls back to this value.
- `donors` — a list of all threads whose priority has been donated *to this thread*. A thread can receive donations from multiple waiters across multiple locks.
- `donor_elem` — a **separate** `list_elem` from the existing `elem`. You cannot reuse `elem` because a blocked thread's `elem` is already on a semaphore wait list. `donor_elem` is used exclusively for the `donors` list.
- `waiting_on_lock` — the key that links a donor to a specific lock. During `lock_release()` you iterate `donors` and remove every donor whose `waiting_on_lock` matches the released lock. It also enables walking the nested donation chain upward.

**Step 2 — Initialize new fields in** `init_thread()`

In `src/threads/thread.c`, inside `init_thread()`, add:

```c
t->original_priority = priority;
list_init (&t->donors);
t->waiting_on_lock = NULL;
```

Every new thread starts with no donors, no waiting lock, and an `original_priority` that matches the `priority` argument.

**Step 3 — Implement priority donation in** `lock_acquire()` **(**`src/threads/synch.c`**)**

Before the existing `sema_down()` call, insert donation logic:

1. Record what we're waiting on:

```c
struct thread *cur = thread_current ();
cur->waiting_on_lock = lock;
```

1. If the lock is already held, donate our priority up the chain:

```c
if (lock->holder != NULL)
  {
    list_insert_ordered (&lock->holder->donors, &cur->donor_elem,
                         thread_priority_greater, NULL);

    struct lock *l = lock;
    int depth = 0;
    while (l != NULL && l->holder != NULL && depth < 8)
      {
        if (cur->priority > l->holder->priority)
          l->holder->priority = cur->priority;
        l = l->holder->waiting_on_lock;
        depth++;
      }
  }
```

Concrete trace through `priority-donate-nest`:

- Thread L (pri 31) holds lock A.
- Thread M (pri 32) acquires lock B, then tries lock A → blocked. M donates 32 to L. Chain: `lock A → holder L`. L becomes 32.
- Thread H (pri 33) tries lock B → blocked. H donates 33 to M. Chain walk: `lock B → holder M → waiting_on_lock (lock A) → holder L`. Both M and L are boosted to 33.

1. After `sema_down()` returns (we now own the lock):

```c
cur->waiting_on_lock = NULL;
lock->holder = cur;
```

You should disable interrupts around the donation section (before `sema_down()`) to prevent a timer interrupt from seeing inconsistent state:

```c
enum intr_level old_level = intr_disable ();
cur->waiting_on_lock = lock;
/* donation chain walk here */
intr_set_level (old_level);
sema_down (&lock->semaphore);
old_level = intr_disable ();
cur->waiting_on_lock = NULL;
lock->holder = cur;
intr_set_level (old_level);
```

**Step 4 — Implement the priority recomputation helper (**`src/threads/thread.c`**)**

```c
void
thread_recompute_priority (struct thread *t)
{
  t->priority = t->original_priority;
  if (!list_empty (&t->donors))
    {
      struct thread *top_donor = list_entry (list_front (&t->donors),
                                             struct thread, donor_elem);
      if (top_donor->priority > t->priority)
        t->priority = top_donor->priority;
    }
}
```

This works because `donors` is kept sorted in descending priority order (highest-priority donor at the front via `list_insert_ordered`). Checking the front element is O(1).

Declare this function in `thread.h`:

```c
void thread_recompute_priority (struct thread *);
```

**Step 5 — Implement donor removal in** `lock_release()` **(**`src/threads/synch.c`**)**

Before the existing `sema_up()` call, remove all donors that were waiting on the lock being released, then recalculate priority:

```c
struct thread *cur = thread_current ();

struct list_elem *e = list_begin (&cur->donors);
while (e != list_end (&cur->donors))
  {
    struct thread *t = list_entry (e, struct thread, donor_elem);
    if (t->waiting_on_lock == lock)
      e = list_remove (e);
    else
      e = list_next (e);
  }

thread_recompute_priority (cur);

lock->holder = NULL;
sema_up (&lock->semaphore);
```

Trace through `priority-donate-multiple`:

- Main (pri 31) holds locks A and B.
- Thread a (pri 34) donates via lock A. Thread b (pri 36) donates via lock B. Main's effective priority = 36.
- Main releases lock B: remove donors where `waiting_on_lock == &b` → thread b removed. Recompute: `max(31, 34) = 34`.
- Main releases lock A: remove thread a. Recompute: `max(31) = 31`. Main drops to base.

Trace through `priority-donate-multiple2`:

- Main (pri 31) holds locks A and B. Thread a (pri 34) donates via A. Thread b (pri 36) donates via B. Thread c (pri 32) has no lock, just sits on the ready list.
- Main releases lock A first: remove thread a. Recompute: `max(31, 36) = 36`. Main stays at 36, so thread c (pri 32) still doesn't run.
- Main releases lock B: remove thread b. Recompute: 31. Thread b runs (36), then thread a runs (34), then thread c runs (32), then main finishes.

**Step 6 — System-wide queue enforcement**

This is not a single location — it is a principle enforced everywhere:

- Every insertion into `ready_list` uses `list_insert_ordered()` by priority (already done in Task 2).
- Every insertion into `sema->waiters` uses `list_insert_ordered()` by priority (already done in Task 2).
- In `sema_up()`, use `list_max()` to find the highest-priority waiter rather than `list_pop_front()`, because a thread's priority may have changed (via donation) after it was inserted. `list_max()` always picks the current highest.
- In `cond_signal()`, find the `semaphore_elem` whose waiter has the highest priority.
- After any priority change (donation, release, `thread_set_priority()`), check if preemption is needed.

### Things to Consider

- **Why a separate** `donor_elem`**?** The existing `elem` in `struct thread` is already in use: when READY it sits on `ready_list`; when BLOCKED it sits on a semaphore's `waiters` list. A donating thread is BLOCKED and needs to be on *both* a semaphore wait list (via `elem`) and a holder's donor list (via `donor_elem`). Using the same `list_elem` for both would corrupt one of the lists.
- **Nested donation depth limit = 8.** The `priority-donate-chain` test creates exactly `NESTING_DEPTH = 8` levels. Your propagation loop must iterate at most 8 times. Without this limit, a pathological lock chain could cause unbounded time spent in an interrupt-disabled critical section.
- **Multiple donations from different locks.** A thread can hold multiple locks simultaneously (`priority-donate-multiple`). When you release one lock, you must only remove donors for *that specific lock* — not flush the entire donor list. You identify which donors belong to which lock by checking `donor->waiting_on_lock`.
- **Multiple donations to the same lock.** In `priority-donate-one`, two threads (pri 32 and 33) both wait on the same lock held by main (pri 31). Both donate. When the lock is released, both donors are removed and main returns to 31.
- `priority-donate-lower` **and** `thread_set_priority()` **interaction.** When a thread has an active donation of 41 and calls `thread_set_priority(21)`, only `original_priority` changes to 21. The effective priority remains 41 until the lock is released. `thread_set_priority()` must call `thread_recompute_priority()`.
- `priority-donate-sema` **shows donation + semaphore interaction.** Thread L holds a lock then blocks on a semaphore. Thread H tries the lock and donates to L. When L is woken by `sema_up()`, it runs at H's donated priority, releases the lock (waking H), and only then drops back. Donation works correctly even when the holder is blocked on something other than the donated lock.
- **Interrupt safety around donation logic.** `lock_acquire()` is called with interrupts enabled. Your donation code (adding to `donors`, walking the chain) runs *before* `sema_down()`. Disable interrupts around this section to prevent a timer interrupt from seeing an inconsistent donor list or partially-updated priorities.
- **Don't donate through semaphores or condition variables** — only through locks. Semaphores have no `owner` field, so there is no target to donate to. The tests only require lock-based donation.
- `struct thread` **size budget.** The new fields add approximately 24 bytes (`int` = 4 + `struct list` = 8 + `struct list_elem` = 8 + pointer = 4). The original struct is 72 bytes. At 96 bytes you still have plenty of room within the 4KB page.
- **Re-sorting wait lists after donation.** When a thread's priority changes via donation, it might already be on a semaphore `waiters` list at its old priority. In `sema_up()`, using `list_max()` instead of `list_pop_front()` is safer because it always finds the current highest-priority waiter regardless of insertion order.
- **Order of operations in** `lock_release()`**.** Set `lock->holder = NULL` *before* `sema_up()`. If you set it after, the thread that `sema_up()` unblocks might immediately run (preemption) and try to access the lock's holder, seeing stale data. The original code already does this — make sure your new donor-removal code doesn't reorder it.
- **When walking the donation chain, use** `cur->priority` **not** `original_priority`**.** Thread H donating to a chain should propagate its *effective* priority (which might itself be a donated priority in theory, though the tests don't exercise this).

---

## Task 4 — `thread_set_priority` / `thread_get_priority`

**Goal:** Allow a thread to examine and modify its own priority, respecting active donations.

### Files to Modify


| File                   | What to Do                                                   |
| ---------------------- | ------------------------------------------------------------ |
| `src/threads/thread.c` | Rewrite `thread_set_priority()` and `thread_get_priority()`. |


### Files to Read / Understand First


| File                                        | Why                                                                                                 |
| ------------------------------------------- | --------------------------------------------------------------------------------------------------- |
| `src/threads/thread.c`                      | See the current stubs at lines 335–346.                                                             |
| `src/tests/threads/priority-change.c`       | Verifies that lowering your own priority causes preemption.                                         |
| `src/tests/threads/priority-donate-lower.c` | Verifies that lowering base priority while a donation is active does not reduce effective priority. |


### Step-by-Step Plan

**Step 1 — Rewrite** `thread_set_priority()`

Key points:

- Only `original_priority` is set to `new_priority`. The effective `priority` is then recalculated via `thread_recompute_priority()`, which takes the max of `original_priority` and the highest active donation.
- After the change, we check whether to yield. If a thread in the ready list has higher priority, we voluntarily give up the CPU.

**Step 2 — Rewrite** `thread_get_priority()`



This function is actually unchanged from the original skeleton. The `priority` field is already the *effective* priority once donation logic is in place. No extra work is needed here.

### Things to Consider

- **Lowering priority below a donation.** If a thread has a donated priority of 50 and calls `thread_set_priority(10)`, the effective priority stays 50. Only `original_priority` changes to 10. After the donation is released, the thread drops to 10. The `priority-donate-lower` test checks exactly this scenario.
- **Raising priority above all donations.** If a thread has a donated priority of 40 and calls `thread_set_priority(50)`, `thread_recompute_priority()` sets effective priority to `max(50, 40) = 50`. The thread keeps running at 50.
- **Yield check is essential.** The `priority-change` test creates a higher-priority thread, then the main thread lowers its own priority. Without the yield check, the main thread would keep running even though a higher-priority thread is ready.
- **No interrupt disable needed for the simple case.** `thread_set_priority()` only modifies the calling thread's own fields. Since only the running thread calls it (not an interrupt handler), there is no race on `original_priority`. However, if your `thread_recompute_priority()` accesses the `donors` list, you may want to disable interrupts briefly during the recomputation to prevent a concurrent interrupt from modifying the list.
- **MLFQS guard.** The advanced scheduler (MLFQS) does not use manual priority setting. If you later implement MLFQS, `thread_set_priority()` should be a no-op when `thread_mlfqs` is true. For now this is not required, but keep it in mind.

