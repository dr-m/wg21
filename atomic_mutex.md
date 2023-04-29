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
resemble `mutex` and `shared_mutex` but can have more desirable
memory characteristics.

# Motivation and Scope

The `mutex` and `shared_mutex` typically wrap synchronization objects
defined by the operating system. For special use cases, such as block
descriptors in a database buffer pool, small storage size and minimal
implementation overhead can be more important than observability or
compatibility with operating system primitives via `native_handle()`.

The proposed `atomic_mutex` and `atomic_shared_mutex` address the
following shortcomings of `mutex` and `shared_mutex`:

For historical reasons, `mutex` and `shared_mutex` can be larger than
necessary. For example, the size of `pthread_mutex_t` is 48 bytes on
64-bit GNU/Linux. In a prototype implementation, we have a 4-byte
`atomic_mutex` and an 8-byte `atomic_shared_mutex`.

A small mutex could be embedded deep in concurrent data structures. An
extreme could be to have one mutex per CPU cache line, covering a
number of data items in the same cache line. This would reduce false
sharing and improve the locality of reference: As a byproduct of
acquiring a mutex, some data protected by the mutex can be loaded to
the cache. For example, a hash table can have mutexes interleaved with
pointers to the hash bucket chains.

## Compatibility with memory transactions

There does not appear to be a way to flexibly use lock elision with
`mutex` or `shared_mutex`. There is a global parameter
[@glibc.elision] that could apply to every `std::mutex::lock()` call.
But, lock elision can only work if the critical section is small and
free from any system calls. Failed lock elision attempts hurt
performance. To allow lock elision to be implemented in a hardware
memory transaction, we propose the member functions `is_locked()` and
`is_locked_or_waiting()`.

While memory transactions are out of the scope of this proposal, we
feel that they are important enough to be accounted for.

Implementing lock elision in a memory transaction requires some
predicates. For `mutex` or `shared_mutex` or typical operating system
mutexes, no predicates like `is_locked()` or `is_locked_or_waiting()`
are defined, possibly because using them outside assertions may be
considered bad style.

Our proposal makes the member functions `is_locked()` and
`is_locked_or_waiting()` available as protected member functions of
`atomic_mutex` and `atomic_shared_mutex`. The protected instead of
public access should discourage improper usage in applications.

The prototype in [@atomic_sync] includes a `transactional_lock_guard`
that resembles `std::lock_guard` but supports a compile-time option
for supporting lock elision by using a hardware memory transaction when
the capability is detected during runtime.

## Spin-loops; use with `lock_guard`

In high-contention applications where locks are expected to be held
for a very short time, one may want to hint that in case a lock cannot
be granted instantly, it would be desirable to always attempt busy-waiting
for some duration for it to become available (spin-loop), before
suspending the execution of the thread. This could avoid the execution
of expensive system calls in a multi-threaded system.

There can exist implementation-defined attributes for enabling
spin-loops, but there does not appear to be a way to specify such
attributes when constructing a `std::mutex` or
`std::shared_mutex`. Moreover, a spin-loop like [@glibc.spin] would
affect all `lock()` or `shared_lock()` operations on a particular
lock, or all locks in the program, which is not practical.

Because the efficient implementation of spin-loops depends on
application and hardware architecture, they are out of the scope of
this proposal. A programmer might want to define convenience classes
to enable spinning for some acquisitions, or for every acquisition,
something like the following:
```cpp
#include <emmintrin.h>
struct spin_capable_mutex : public std::atomic_mutex
{
  void spin_lock() { while (!try_lock()) while (is_locked()) _mm_pause(); }
};
struct spin_mutex : public spin_capable_mutex
{
  void lock() noexcept { spin_lock(); }
};
```

# Impact on the Standard

This proposal is a pure library extension.

# Proposed Wording

Add after __§33.6.3 [shared.mutex.syn]__ the subsection
[atomic.mutex.syn] "Header `<atomic_mutex>` synopsis":

```cpp
namespace std {
  // [thread.mutex.atomic], class atomic_mutex
  class atomic_mutex;
};
```

Add after __[atomic.mutex.syn]__ the subsection [atomic.shared.mutex.syn]
"Header `<atomic_shared_mutex>` synopsis"
with the following contents:
```cpp
namespace std {
  // [thread.sharedmutex.atomic], class atomic_shared_mutex
  class atomic_shared_mutex;
};
```

Change the end of the first sentence of
__§33.6.4.2.1 [thread.mutex.requirements.mutex.general]__

>… `shared_mutex`, and `shared_timed_mutex`.

to

>… `shared_mutex`, `shared_timed_mutex`, `atomic_mutex`,
>and `atomic_shared_mutex`.

Add after __§33.6.4.3.3 [thread.mutex.recursive]__        the section
[thread.mutex.atomic] "Class `atomic_mutex`":
```cpp
namespace std {
  class atomic_mutex {
    // exposition only
    atomic_unsigned_lock_free a = 0;
  protected:
    bool is_locked() const noexcept { return a; }
    bool is_locked_or_waiting() const noexcept { return a; }
  public:
    bool try_lock() noexcept { return !a.exchange(1); }
    void lock() noexcept { while(!try_lock()) a.wait(1); }
    void unlock() noexcept { a = 0; a.notify_one(); }
  }
};
```
>The class `atomic_mutex` provides a non-recursive mutex with
>exclusive ownership semantics. If one thread owns an `atomic_mutex`
>object, attempts by another thread to acquire ownership of that
>object will fail (for `try_lock()`) or block (for `lock()`)
>until another thread has released ownership with a call to `unlock()`.

>[Note 1: `atomic_mutex` will not keep track of its owning thread. It
>is not an error to acquire ownership in one thread and release it
>in another. — end note]

>The class `atomic_mutex` meets all of the mutex requirements
>([thread.mutex.requirements]).  It is a standard-layout class
>([class.prop]).

>[Note 2: A program will deadlock if the thread that owns a mutex
>object calls `lock()`` on that object. If the implementation can
>detect the deadlock, a
>`resource_deadlock_would_occur` error condition might be observed. —
>end note]

>The behavior of a program is undefined if it destroys an
>`atomic_mutex` object owned by any thread.

Add after __[thread.mutex.atomic]__ the section
[thread.sharedmutex.atomic] "Class `atomic_shared_mutex`":
```cpp
namespace std {
  class atomic_shared_mutex {
    // exposition only
    atomic_unsigned_lock_free inner = 0;
    atomic_mutex outer;
    static constexpr unsigned X = ~(~0U >> 1);

    void lock_inner_wait(unsigned lk) noexcept
    {
      lk |= X;
      do {
        inner.wait(lk);
        lk = inner.load(std::memory_order_acquire);
      }
      while (lk != X);
    }

    void lock_inner() noexcept
    {
      if (unsigned lk = inner.fetch_or(X))
        lock_inner_wait(lk);
    }

  protected:
    bool is_locked() const noexcept { return inner == X; }
    bool is_locked_or_waiting() const noexcept
    { return is_locked() || outer.is_locked_or_waiting(); }
  public:
    bool try_lock() noexcept
    {
      if (!outer.try_lock())
        return false;
      lock_inner();
      return true;
    }
    void lock() noexcept { outer.lock(); lock_inner(); }
    void unlock() noexcept { inner = 0; outer.unlock();  }

    bool try_lock_shared() noexcept
    {
      unsigned lk = 0;
      using o = std::memory_order;
      while (!inner.compare_exchange_weak(lk, lk + 1, o::acquire, o::relaxed))
        if (lk & X)
          return false;
      return true;
    }
    void lock_shared() noexcept
    {
      if (!try_lock_shared()) {
        bool acquired;
        do {
          outer.lock();
          acquired = try_lock_shared();
          outer.unlock();
        } while (!acquired);
      }
    }
    void unlock_shared() noexcept { if (--inner == X) inner.notify_one(); }

    bool try_lock_update() noexcept
    {
      if (!outer.try_lock())
        return false;
      inner++;
      return true;
    }
    void lock_update() noexcept { outer.lock(); inner++; }
    void unlock_update() noexcept { inner--; outer.unlock(); }
    void update_lock_downgrade() noexcept { inner = 1; }
    void update_lock_upgrade() noexcept
    {
      if (unsigned lk = inner.fetch_add(X - 1) - 1)
        lock_inner_wait(lk);
    }
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

>In addition to `shared_lock()`, there are two non-shared lock operations:
>the exclusive `lock()` and an `update_lock()` that conflicts with itself
>and `lock()` but allows a number of `shared_lock()` to be held concurrently.
>The function `update_lock_upgrade()` upgrades a granted `update_lock()`
>to `lock()`, and `update_lock_downgrade()` converts an exclusive `lock()`
>to `update_lock()`.

# Design Decisions

The `atomic_mutex` is intended to be a plug-in replacement of `mutex`.

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

An implementation may allow zero-initialized memory to be interpreted
as a valid object. Clang starting with version 3.4.1 as well as GCC
starting with version 10 seem to be able to emit calls to `memset()`
when an array is allocated from the stack, and the `constexpr`
constructor is zero-initializing the object.

We might want to leave room for implementations where the internal
representation of an unlocked `atomic_mutex` or `atomic_shared_mutex`
object is not zero-initialized.

# Implementation Experience

An refined implementation of `atomic_mutex` and `atomic_shared_mutex`
exists. It has been tested on GNU/Linux on IA-32, AMD64, ARMv7,
ARMv8, POWER 8, s390x with various versions of GCC between 4.8.5 and
12.2.0, Clang (including versions 9, 12, 13, 15), as well as with the
latest Microsoft Visual Studio on Microsoft Windows on AMD64.

The implementation also supports C++11 by emulating the C++20
`atomic::wait()` and `atomic::notify_one()` via `futex` system calls
on Linux or OpenBSD.

The prototype implementation [@atomic_sync] is based on a C++11
implementation that is part of [@mariadb_server] starting with version 10.6,
using futex-like operations on Linux, OpenBSD, FreeBSD, DragonFly BSD
and Microsoft Windows. It includes an interface to a data race detector
[@tsan], which passes some tests on Clang 14, 15, and GCC 12.

The `transactional_lock_guard` has been tested on GNU/Linux on POWER 8
(with runtime detection of the POWER v2.09 Hardware Transactional Memory) and
on IA-32 and AMD64 (with runtime detection of Intel TSX-NI a.k.a. RTM).
It has also been tested on Microsoft Windows, but only on a processor
that does not support TSX-NI or RTM.

# Future Work

Because `condition_variable` and `condition_variable_any` do not work
with non-exclusive modes of `shared_mutex` or `atomic_shared_mutex`,
we might want to introduce an `atomic_condition_variable`.
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
  - id: tsan
    citation-label: ThreadSanitizer
    title: "Thread Sanitizer manual"
    URL: https://github.com/google/sanitizers/wiki/ThreadSanitizerCppManual
---
