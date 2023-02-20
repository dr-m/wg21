---
title: "Transparent mutex and shared_mutex based on atomic"
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
implementation overhead may be more important than observability or
compatibility with operating system primitives via `native_handle()`.

The proposed `atomic_mutex` and `atomic_shared_mutex` address the
following shortcomings of `mutex` and `shared_mutex`:

For historical reasons, `mutex` and `shared_mutex` may be larger than
necessary. For example, the size of `pthread_mutex_t` is 48 bytes on
64-bit GNU/Linux. In a prototype implementation, we have a 4-byte
`atomic_mutex` and an 8-byte `atomic_trans_mutex`.

A small mutex could be embedded deep in concurrent data structures. An
extreme could be to have one mutex per CPU cache line, covering a
number of data items in the same cache line. This would reduce false
sharing and improve the locality of reference: As a byproduct of
acquiring a mutex, some data protected by the mutex may be loaded to
the cache. For example, a hash table may have mutexes interleaved with
pointers to the hash bucket chains.

Application developers may know best when to use a spin-loop when
acquiring a lock. Spinning may be useful (avoiding context switches)
on systems that employ symmetrical multiprocessing
when a mutex is contended and the critical sections are small. But,
spinning could be harmful if we are holding another lock that would
prevent other threads from attempting to acquire the lock that we are
trying to acquire, or when attempts to acquire the lock would cause
excessive cache line invalidation between processor cores.
We propose member functions like `spin_lock()` that
are like `lock()`, but may include a spin-loop before entering a wait.

There may exist implementation-defined attributes for enabling
spin-loops, but there does not appear to be a way to specify such
attributes when constructing a `std::mutex` or
`std::shared_mutex`. Moreover, a spin-loop like [@glibc.spin] would
affect all `lock()` or `shared_lock()` operations on a particular
lock, or all locks in the program, which is not practical.

There does not appear to be a way to flexibly use lock elision with
`mutex` or `shared_mutex`. There is a global parameter
[@glibc.elision] that could apply to every `std::mutex::lock()` call.
But, lock elision can only work if the critical section is small and
free from any system calls. Failed lock elision attempts hurt
performance.

To allow maximum transparency, the proposed `atomic_mutex` and
`atomic_shared_mutex` allow an underlying storage class to be defined
or extended by the user. This allows an application developer to
control the exact storage size and format, as well as to define
nonstandard predicates like `is_locked()` or `is_locked_or_waiting()`.
The default storage is derived from `std::atomic` on an integer type.

## atomic_shared_mutex

A prototype implementation `atomic_shared_mutex` in [@atomic_sync]
additionally supports an `update_lock()` operation that conflicts with
itself and `lock()` but allows concurrent `shared_lock()`. The
`update_lock()` could allow more concurrency in case some parts of the
covered data structure are never read under `shared_lock()`.

## Use with `lock_guard`

In high-contention applications where locks are expected to be held
for a very short time, one may want to hint that in case a lock cannot
be granted instantly, it would be desirable to always attempt busy-waiting
for some duration for it to become available (spin-loop), before
suspending the execution of the thread. This could avoid the execution
of expensive system calls in a multi-threaded system.

If a lock is expected to be held for longer time, it may be better to
avoid any spin-loop and suspend the execution of the thread
immediately. This could be a reasonable default for `atomic_mutex` and
`atomic_shared_mutex`.

By default, `lock_guard` on `atomic_mutex` or `atomic_shared_mutex`
would invoke the regular `lock()` member function, which does not
explicitly request a spin-loop to be executed. An application might
want to define convenience classes to enable spinning for every
acquisition. A possible implementation could be as follows:
```cpp
class spin_mutex : public std::atomic_mutex
{
public:
  void lock() { spin_lock(); }
};

class spin_shared_mutex : public std::atomic_shared_mutex
{
public:
  void lock() { spin_lock(); }
  void shared_lock() { spin_lock_shared(); }
};
```

## Compatibility with memory transactions

While memory transactions are out of the scope of this proposal, we
feel that they are important enough to be accounted for.

Implementing lock elision in a memory transaction requires some
predicates. For `mutex` or `shared_mutex` or typical operating system
mutexes, no predicates like `is_locked()` or `is_locked_or_waiting()`
are defined, possibly because using them outside assertions may be
considered bad style.

The prototype in [@atomic_sync] includes a `transactional_lock_guard`
that resembles `std::lock_guard` but supports lock elision by using
a memory transaction when support is enabled during compilation time
and detected during runtime.  The predicates `is_locked()` and
`is_locked_or_waiting()` that are necessary for lock elision are
defined in `transactional_mutex_storage` and
`transactional_shared_mutex_storage` classes that are derived from
`atomic_mutex_storage<>` and `shared_mutex_storage<>`. The hardware
lock elision works with `atomic_mutex<transactional_mutex_storage>`
and `atomic_shared_mutex<transactional_shared_mutex_storage>`.

# Impact on the Standard

This proposal is a pure library extension.

# Proposed Wording

Add after __§33.6.3 [shared.mutex.syn]__ the subsection
[atomic.mutex.syn] "Header `<atomic_mutex>` synopsis":

```cpp
namespace std {
  // [thread.mutex.atomic.storage], class atomic_mutex_storage
  template<typename T = uint32_t>
  class atomic_mutex_storage : public std::atomic<T>;

  // [thread.mutex.atomic], class atomic_mutex
  template<typename storage = atomic_mutex_storage<>>
  class atomic_mutex : public storage;
};
```

Add after __[atomic.mutex.syn]__ the subsection [atomic.shared.mutex.syn]
"Header `<atomic_shared_mutex>` synopsis"
with the following contents:
```cpp
namespace std {
  template<typename T = uint32_t>
  class atomic_shared_mutex_storage : public atomic_mutex_storage<T>;

  // [thread.sharedmutex.atomic], class atomic_shared_mutex
  template<typename storage = atomic_shared_mutex_storage<>>
  class atomic_shared_mutex : public storage;
};
```

Change the end of the first sentence of
__§33.6.4.2.1 [thread.mutex.requirements.mutex.general]__
>… `shared_mutex`, and `shared_timed_mutex`.
to
>… `shared_mutex`, `shared_timed_mutex`, `atomic_mutex`, `atomic_shared_mutex`.

Add after __§33.6.4.3.3 [thread.mutex.recursive] the section
[thread.mutex.atomic]` "Class `atomic_mutex`":
```cpp
namespace std {
  template<typename T = uint32_t>
  class atomic_mutex_storage : public std::atomic<T> {
  public:
    using type = T;
    static constexpr type HOLDER = type(~(type(~type(0)) >> 1));
    static constexpr type WAITER = 1;

    static unsigned spin_rounds;

    void wait_and_lock();
    void spin_wait_and_lock();

    // for atomic_shared_mutex
    void lock_wait(T lk);
  };

  template<typename storage = atomic_mutex_storage<>>
  class atomic_mutex : public storage {
  public:
    constexpr atomic_mutex();
    constexpr ~atomic_mutex();
    atomic_mutex(const atomic_mutex&) = delete;
    atomic_mutex& operator=(const atomic_mutex&) = delete;

    bool try_lock() noexcept;
    void lock();
    void unlock() noexcept;

    void spin_lock();
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
>involve busy-waiting (spin-loop) if the object is owned by any
>thread.

>The number of loops in `spin_lock()` may be controlled by
>defining `storage::spin_rounds`. **FIXME: How to word this?**

>[Note 2: A program will deadlock if the thread that owns a mutex
>object calls `lock()` or `spin_lock()` on that object.  If the
>implementation can detect the deadlock, a
>`resource_deadlock_would_occur` error condition might be observed. —
>end note]

>The behavior of a program is undefined if it destroys an
>`atomic_mutex` object owned by any thread.

Add after __[thread.mutex.atomic]__ the section
[thread.sharedmutex.atomic]` "Class `atomic_shared_mutex`":
```cpp
namespace std {

  template<typename T = uint32_t>
  class atomic_shared_mutex_storage : public atomic_mutex_storage<T> {
    atomic_mutex<mutex_storage<T>> ex;
  };

  template<typename storage = atomic_shared_mutex_storage<>>
  class atomic_shared_mutex : public storage {
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
>(spin-loop) if the object is owned by any thread.

# Design Decisions

The `atomic_mutex` is intended to be a plug-in replacement of `mutex`.

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
zero-initialized memory to be interpreted as a valid object.  Clang
starting with version 3.4.1 as well as GCC starting with version 10
seem to be able to emit calls to `memset()` when an array is allocated
from the stack, and the `constexpr` constructor is zero-initializing
the object.

We may want to leave room for implementations where the internal
representation of an unlocked `atomic_mutex` or `atomic_shared_mutex`
object is not zero-initialized.

# Implementation Experience

An implementation of `atomic_mutex` and `atomic_shared_mutex` exists.
It has been tested on GNU/Linux on IA-32, AMD64, ARMv7, ARMv8, POWER
8, s390x with various versions of GCC between 4.8.5 and 12.2.0, Clang
(including versions 9, 12, 13, 15), as well as with the latest
Microsoft Visual Studio on Microsoft Windows on AMD64.

The implementation also supports C++11 by emulating the C++20
`atomic::wait()` and `atomic::notify_one()` via `futex` system calls
on Linux or OpenBSD.

The prototype implementation [@atomic_sync] is based on a C++11
implementation that is part of [@mariadb_server] starting with version 10.6,
using futex-like operations on Linux, OpenBSD, FreeBSD, DragonFly BSD
and Microsoft Windows.
That code base also includes a thin wrapper of Microsoft `SRWLOCK`
(`sizeof(SRWLOCK)==sizeof(size_t)`, that is, 4 or 8 bytes)
for those cases where `update_lock()` is not needed.
On operating systems for which a futex-like interface has not been
implemented, the wait queues that futexes would provide are simulated
with mutexes and condition variables, which incurs some storage overhead.

The `transactional_lock_guard` has been tested on GNU/Linux on POWER 8
(with runtime detection of the POWER v2.09 Hardware Transactional Memory) and
on IA-32 and AMD64 (with runtime detection of Intel TSX-NI a.k.a. RTM).

The `transactional_lock_guard` implementation has also been tested on
Microsoft Windows, but only on a processor that does not support
TSX-NI or RTM.

# Future Work

A variant of `atomic_shared_mutex` that includes `update_lock()` could
be defined, either as part of this proposal or as a future proposal.

Because `condition_variable` and `condition_variable_any` do not work
with non-exclusive modes of `shared_mutex` or `atomic_shared_mutex`,
we might want to introduce a `atomic_condition_variable`.
However, a straightforward implementation of `wait_until()` would
require `atomic::wait_until()` to be defined. A prototype
implementation `atomic_condition_variable` without `wait_until()` is
available in [@atomic_sync].

It might be useful to introduce a `transactional_lock_guard` to
simplify the implementation of lock elision.

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
  - id: glibc.spin
    citation-label: glibc.spin
    title: "GNU libc POSIX Thread Tunables"
    URL: https://www.gnu.org/software/libc/manual/html_node/POSIX-Thread-Tunables.html
  - id: glibc.elision
    citation-label: glibc.elision
    title: "GNU libc Elision Tunables"
    URL: https://www.gnu.org/software/libc/manual/html_node/Elision-Tunables.html
---
