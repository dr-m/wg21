---
title: "Slim mutexes and locks"
subtitle: "`atomic_mutex`, `atomic_shared_mutex`"
document: D0000R0
date: today
audience:
  - Library Evolution Group
author:
  - name: Marko Mäkelä
    email: <marko.makela@iki.fi>
toc: true
toc-depth: 2
---

# Introduction

This paper proposes `atomic_mutex` and `atomic_shared_mutex` that
resemble `mutex` and `shared_mutex` but may have more desirable
memory characteristics.

# Motivation and Scope

The `mutex` and `shared_mutex` typically wrap synchronization objects
defined by the operating system. For special use cases, such as block
descriptors in a database buffer pool, small storage size and minimal
implementation overhead are more important than compatibility with
operating system primitives via `native_handle()`.

The proposed `atomic_mutex` and `atomic_shared_mutex` address the
following shortcomings of `mutex` and `shared_mutex`:

For historical reasons, `mutex` and `shared_mutex` may be larger than
necessary. For example, `sizeof(pthread_mutex_t)==48` on 64-bit
GNU/Linux. For a prototype implementation, we have
`sizeof(atomic_mutex)==4` and `sizeof(atomic_shared_mutex)==8`. A
small mutex could be embedded deep in concurrent data structures. An
extreme could be to have one mutex per CPU cache line, covering a
number of data items in the same cache line. This would reduce false
sharing and improve the locality of reference: As a byproduct of
acquiring a mutex, some data protected by the mutex may be loaded to
the cache. For example, a hash table may have mutexes interleaved with
pointers to the hash bucket chains.

Application developers may know best when to use a spinloop when
acquiring a lock. Spinning may be useful (avoiding context switches)
when a mutex is contended and the critical sections are small. But,
spinning could be harmful if we are holding another lock that would
prevent other threads from attempting to acquire the lock that we are
trying to acquire. We propose member functions like `spin_lock()` that
are like `lock()`, but may include a spinloop before entering a wait.

There does not appear to be a way to flexibly use lock elision with
`mutex` or `shared_mutex`. There is a global parameter
[@glibc.elision] that could apply to every `std::mutex::lock()` call.
But, lock elision can only work if the critical section is small and
free from any system calls. Failed lock elision attempts hurt
performance.

Implementing lock elision in a memory transaction requires predicates
like `is_locked()`, which are not defined for `mutex` or
`shared_mutex`.

Motivated by practical needs, our proposed `atomic_shared_mutex`
extends `shared_mutex` with a new operation `update_lock()` that is
mutually exclusive with itself and `lock()` but not `shared_lock()`.

## atomic_mutex

Unlike `mutex` and `shared_mutex`, the locks in this proposal are
specified not to be recursive or re-entrant: If `atomic_mutex::lock()`
has returned to any thread, further calls to `atomic_mutex::lock()`
will not return until `atomic_mutex::unlock()` has been invoked in
some thread.

If an implementation defines a destructor, its behavior is undefined if
`is_locked_or_waiting()` holds, that is, if the `atomic_mutex` is
owned or being waited for by any thread or if any thread terminates
while holding any ownership of or waiting for the `atomic_mutex`.

## atomic_shared_mutex

The proposed `atomic_shared_mutex` is mostly like `shared_mutex`, with
the difference that it defines the predicates `is_locked()` and
`is_locked_or_waiting()`.

A prototype implementation [@atomic_sync] supports an additional
`update_lock()` mode that conflicts with itself and `lock()` but
allows concurrent `shared_lock()`. The `update_lock()` could allow
more concurrency in case some parts of the covered data structure are
never read under `shared_lock()`. The `update_lock()` could also cover
a write of a database page into the file system.

## Use with `lock_guard`

In high-contention applications where locks are expected to be held
for a very short time, one may want to hint that in case a lock cannot
be granted instantly, it would be desirable to always attempt busy-waiting
for some duration for it to become available (spinloop), before
suspending the execution of the thread. This could avoid the execution
of expensive system calls in a multithreaded system.

If a lock is expected to be held for longer time, it may be better to
avoid any spinloop and suspend the execution of the thread
immediately. This could be a reasonable default for `atomic_mutex` and
`atomic_shared_mutex`.

By default, `lock_guard` on `atomic_mutex` or `atomic_shared_mutex`
would invoke the regular `lock()` member function, which does not
explicitly request a spinloop to be executed. An application might
want to define convenience classes to enable spinning for every
acquisition. A possible implementation could be as follows:
```cpp
class atomic_spin_mutex : public atomic_mutex
{
public:
  void lock() { spin_lock(); }
};

class atomic_spin_shared_mutex : public atomic_shared_mutex
{
public:
  void lock() { spin_lock(); }
  void shared_lock() { spin_lock_shared(); }
#ifdef UPDATE_LOCK
  void update_lock() { spin_lock_update(); }
#endif
};
```

# Impact on the Standard

This proposal is a pure library extension.

# Proposed Wording

Add after __§32.5.3 [shared.mutex.syn]__ the subsections
[atomic.mutex.syn] "Header <atomic_mutex> synopsis":

```cpp
namespace std {
  // [thread.mutex.atomic], class atomic_mutex
  class atomic_mutex;
};
```

Add after __[atomic.mutex.syn]__ the subsection [atomic.shared.mutex.syn]
"Header <atomic_shared_mutex> synopsis"
with the following contents:
```cpp
namespace std {
  // [thread.sharedmutex.atomic], class atomic_shared_mutex
  class atomic_mutex;
};
```

Change the end of the first sentence of
__§32.5.4.1.1 [thread.mutex.requirements.mutex.general]__
>… `shared_mutex`, and `shared_timed_mutex`.
to
>… `shared_mutex`, `shared_timed_mutex`,
>`atomic_mutex`, and `atomic_spin_shared_mutex`.

Add after __§32.5.4.2.3 [thread.mutex.recursive] the section
[thread.mutex.atomic]` "Class `atomic_mutex`":
```cpp
namespace std {
  class atomic_mutex {
  public:
    constexpr atomic_mutex();
    ~atomic_mutex();
    atomic_mutex(const atomic_mutex&) = delete;
    atomic_mutex& operator=(const atomic_mutex&) = delete;

    bool try_lock() noexcept;
    void lock();
    void unlock() noexcept;

    void spin_lock();

    bool is_locked() const noexcept;
    bool is_locked_or_waiting() const noexcept;
  }
};
```
>The class `atomic_mutex` provides a non-recursive mutex with
>exclusive ownership semantics. If one thread owns an `atomic_mutex`
>object, attempts by another thread to acquire ownership of that
>object will fail (for `try_lock()`) or block (for `lock()` or
>`spin_lock()`) until another thread has released ownership with a
>call to `unlock()`.

>[Note 1: `atomic_mutex` will not keep track of its owning thread. It
>is not an error to acquire ownership in one thread and release it
>in another. — end note]

>The class `atomic_mutex` meets all of the mutex requirements
>([thread.mutex.requirements]).  It is a standard-layout class
>([class.prop]).

>The operation `spin_lock()` is similar to `lock()`, but it may
>involve busy-waiting (spinloop) if the object is owned by any
>thread.

>[Note 2: A program will deadlock if the thread that owns a mutex
>object calls `lock()` or `spin_lock()` on that object.  If the
>implementation can detect the deadlock, a
>`resource_deadlock_would_occur` error condition might be observed. —
>end note]

>The predicate `is_locked()` holds on an `atomic_mutex` object
>that is being owned by any thread.

>The predicate `is_locked_or_waiting()` holds on an `atomic_mutex`
>object that is owned by any thread, or a `lock()` or `spin_lock()`
>operation has reached an internal state where a wait is pending.

>The behavior of a program is undefined if it destroys an
>`atomic_mutex` object owned by any thread.

Add after __[thread.mutex.atomic]__ the section
[thread.sharedmutex.atomic]` "Class `atomic_shared_mutex`":
```cpp
namespace std {
  class atomic_shared_mutex {
  public:
    constexpr atomic_shared_mutex();
    atomic_shared_mutex(const atomic_shared_mutex&) = delete;
    atomic_shared_mutex& operator=(const atomic_shared_mutex&) = delete;

    bool try_lock() noexcept;
    void lock();
    void spin_lock();
    void unlock() noexcept;

    bool try_lock_shared() noexcept;
    void lock_shared();
    void spin_lock_shared();
    void unlock_shared() noexcept;

#ifdef UPDATE_LOCK // TBD: Do we want these?
    bool try_lock_update() noexcept;
    void lock_update();
    void spin_lock_update();
    void unlock_update() noexcept;

    void update_lock_upgrade() noexcept;
    void lock_update_downgrade() noexcept;
#endif

    bool is_locked() const noexcept;
    bool is_locked_or_waiting() const noexcept;
  };
}
```
>The class `atomic_shared_mutex` provides a non-recursive mutex with
>shared ownership semantics.

>The class `atomic_shared_mutex` meets all of the shared mutex
>requirements ([thread.sharedmutex.requirements]). It is a
>standard-layout class ([class.prop]).

>The behavior of a program is undefined if:
>1. it destroys an `atomic_shared_mutex` object owned by any thread,
>2. a thread attempts to recursively gain any ownership of an
>`atomic_shared_mutex`, or
>3. a thread terminates while possessing any ownership of an
`atomic_shared_mutex`.

>The operations `spin_lock()` and `spin_lock_shared()` are similar to
>`lock()` and `lock_shared()`, but they may involve busy-waiting
>(spinloop) if the object is owned by any thread.

>The predicate `is_locked()` holds on an `atomic_shared_mutex` object
>that is being exclusively owned by any thread.

>The predicate `is_locked_or_waiting()` holds on an
>`atomic_shared_mutex` object that is owned by any thread, or on which
>a `lock()`, `spin_lock()`, `lock_shared()`, or `spin_lock_shared()`
>operation has reached an internal state where a wait is pending.

FIXME: Do we want to include `update_lock()`? If we omit it,
`atomic_shared_mutex` (as well as `atomic_mutex`) could be a thin
wrapper of the Microsoft Windows `SRWLOCK`, which may perform better
than `std::atomic::wait()` and `std::atomic::notify_one()`.

# Design Decisions

The `atomic_mutex` and `atomic_spin_mutex` are intended to be plug-in
replacements of `mutex`.

For special use cases, such as block descriptors in a database buffer
pool, small storage size and minimal implementation overhead are more
important than compatibility with operating system primitives via
`native_handle()`.

`atomic_shared_mutex::lock_shared()` and
`atomic_shared_mutex::try_lock_shared()` are like `shared_mutex`:
if they are called by a thread that already owns the mutex in any
mode, the behavior is undefined.

The locks are not recursive or re-entrant: If a thread has
successfully acquired a non-shared lock, further calls to acquire a
lock will not return until conflicting locks have been unlocked by any
thread. A lock acquired in Thread _A_ may be unlocked in Thread _B_.
For example, if a database server would acquire a page lock in Thread
_A_ for the purpose of writing it to the file system and later unlock
it in Thread _B_ (after the page has been written to the file system),
a subsequent request to acquire the page lock in Thread _A_ will wait
until Thread _B_ has released the lock.

## Constructors and zero-initialization

A prototype implementation that is based on `atomic<uint32_t>` allows
zero-initialized memory to be interpreted as a valid object.  Clang at
starting with version 3.4.1 as well as GCC starting with version 10
seem to be able to emit calls to `memset()` when an array is allocated
from the stack, and the `constexpr` constructor is zero-initializing
the object.

We may want to leave room for implementations where the internal
representation of an unlocked `atomic_mutex` or `atomic_shared_mutex`
object is not zero-initialized.

# Implementation Experience

An implementation of `atomic_mutex` exists, based on
`atomic<uint32_t>`. An implementation of the `atomic_shared_mutex`
exists, encapsulating `atomic_mutex` and `atomic<uint32_t>`.
It has been tested on GNU/Linux with various versions of GCC between
4.8.5 and 11.2.0, Clang (including versions 9, 12 and 13),
as well as with the latest Microsoft Visual Studio on Microsoft Windows.

The implementation also supports C++11 by emulating the C++20
`atomic::wait()` and `atomic::notify_one()` via `futex` system calls
on Linux or OpenBSD.

The prototype implementation [@atomic_sync] is based on a C++11
implementation that is part of [@mariadb_server] 10.6 (using
futex-like operations on Linux, OpenBSD, and Microsoft Windows).

The prototype also includes a `transactional_lock_guard` that
resembles `std::lock_guard` but supports lock elision by using
transactional memory when support is enabled during compilation time
and detected during runtime.

# Future Work

Because `condition_variable` only works with `mutex` and not
`atomic_mutex`, we might want to introduce `atomic_condition_variable`.
However, a straightforward implementation of  `wait_until()` would
require `atomic::wait_until()` to be defined. A prototype implementation
without `atomic_condition_variable::wait_until()` is available in
[@atomic_sync].

---
references:
  - id: atomic_sync
    citation-label: atomic_sync
    title: "Slim mutex and shared_mutex using C++ std::atomic"
    URL: https://github.com/dr-m/atomic_sync
  - id: mariadb_server
    citation-label: MariaDB Server
    title: "MariaDB: The open source relational database"
    URL: https://github.com/MariaDB/server/
  - id: glibc.elision
    citation-label: glibc.elision
    title: "GNU libc Elision Tunables"
    URL: https://www.gnu.org/software/libc/manual/html_node/Elision-Tunables.html
---
