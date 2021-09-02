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
resemble `mutex` and `shared_mutex` but may have a smaller memory
footprint.

# Motivation and Scope

The `mutex` and `shared_mutex` typically wrap synchronization objects
defined by the operating system.  For special use cases, such as block
descriptors in a database buffer pool, small storage size and minimal
implementation overhead are more important than compatibility with
operating system primitives via `native_handle()`. For historical
reasons, `mutex` and `shared_mutex` may be larger than necessary. For
example, `sizeof(pthread_mutex_t)` is 48 bytes on 64-bit GNU/Linux.

An implementation of `atomic_mutex` exists, based on
`atomic<uint32_t>`. A small mutex could be embedded deep in concurrent
data structures. An extreme could be to have one mutex per CPU cache
line, covering a number of data items in the same cache line. This
would reduce false sharing and improve the locality of reference: As a
byproduct of acquiring a mutex, some data protected by the mutex may
be loaded to the cache. For example, a hash table may have mutexes
interleaved with pointers to the hash bucket chains.

## atomic_mutex

Unlike `mutex` and `shared_mutex`, the locks in this proposal are
specified not to be recursive or re-entrant: If `atomic_mutex::lock()`
has returned to any thread, further calls to `atomic_mutex::lock()`
will not return until `atomic_mutex::unlock()` has been invoked in
some thread.
```cpp
class atomic_mutex
{
public:
  bool trylock() noexcept;
  void lock() noexcept;
  void unlock() noexcept;

  // native_handle_type native_handle();

  // The following are not defined for std::mutex

  /** @return whether the mutex is being held or waited for by any thread */
  bool is_locked_or_waiting() const noexcept;
  /** @return whether the mutex is being held by any thread */
  bool is_locked() const noexcept;
};
```
If an implementation defines a destructor, its behavior is undefined if
`is_locked_or_waiting()` holds, that is, if the `atomic_mutex` is
owned or being waited for by any thread or if any thread terminates
while holding any ownership of or waiting for the `atomic_mutex`.

## atomic_shared_mutex

The proposed `atomic_shared_mutex` is mostly like `shared_mutex`, but
it supports an additional `update_lock()` mode that conflicts with
itself and `lock()` but allows concurrent `shared_lock()`.  The
`update_lock()` could allow more concurrency in case some parts of the
covered data structure are never read under `shared_lock()`. The
`update_lock()` could also cover a write of a database page into the
file system.
```cpp
class atomic_shared_mutex
{
public:
  bool try_lock() noexcept;
  void lock() noexcept;
  void unlock() noexcept;

  bool try_lock_shared() noexcept;
  void lock_shared() noexcept;
  void unlock_shared() noexcept;

  // native_handle_type native_handle();

  // The following are not defined for shared_mutex

  /** Try to acquire an update lock (conflicts with other than shared locks).
  @return whether the Update lock was acquired */
  bool try_lock_update() noexcept;
  /** Acquire an update lock. */
  void lock_update() noexcept;
  /** Release an update lock. */
  void unlock_update() noexcept;

  /** Upgrade an update lock to an exclusive lock.
  (Blocks until unlock_shared() has been invoked on every lock.) */
  void update_lock_upgrade() noexcept;
  /** Downgrade an exclusive lock to an update lock. */
  void lock_update_downgrade() noexcept;

  /** @return whether an exclusive lock is being held or waited for
  by any thread */
  bool is_waiting() const noexcept;
  /** @return whether the exclusive lock is being held by any thread */
  bool is_locked() const noexcept;
  /** @return whether the lock is being held or waited for */
  bool is_locked_or_waiting() const noexcept;
};
```
The proposed `atomic_shared_mutex` is mostly like `shared_mutex`, but
it supports an additional `update_lock()` mode that conflicts with
itself and `lock()` but allows concurrent `shared_lock()`.  The
`update_lock()` could allow more concurrency in case some parts of the
covered data structure are never read under `shared_lock()`. The
`update_lock()` could also cover a write of a database page into the
file system.

## atomic_spin_mutex, atomic_spin_shared_mutex

In high-contention applications where locks are expected to be held
for a very short time, one may want to hint that in case a lock cannot
be granted instantly, it would be desirable to attempt busy-waiting
for some duration for it to become available (spinloop), before
suspending the execution of the thread. This could avoid the execution
of expensive system calls in a multithreaded system.

If a lock is expected to be held for longer time, it may be better to
avoid any spinloop and suspend the execution of the thread
immediately. This could be a reasonable default for `atomic_mutex` and
`atomic_shared_mutex`.

It is up to the implementation to decide whether `atomic_spin_mutex`
and `atomic_spin_shared_mutex` should be treated differently from
`atomic_mutex` and `atomic_shared_mutex`.

# Impact on the Standard

This proposal is a pure library extension.

# Proposed Wording

...

# Design Decisions

The `atomic_mutex` and `atomic_spin_mutex` are intended to be plug-in
replacements of `mutex`.

For special use cases, such as block descriptors in a database buffer
pool, small storage size and minimal implementation overhead are more
important than compatibility with operating system primitives via
`native_handle()`.

`atomic_shared_mutex::lock_shared()` and
`atomic_shared_mutex::try_lock_shared()` are is like `shared_mutex`:
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

To be a plug-in replacement of `mutex` or `shared_mutex`, we must
allow `atomic_mutex` or `atomic_shared_mutex` to be constructed in the
stack. Because stack frames are never guaranteed to be
zero-initialized, this would seem to require the definition of a
constructor that would initialize the memory.

A prototype implementation that is based on `atomic<uint32_t>` allows
zero-initialized memory to be interpreted as a valid object.
In an application that instantiates a large number of `atomic_mutex`
or `atomic_shared_mutex`, it would be desirable to avoid the equivalent of
```cpp
memset(this, 0, sizeof *this);
```
when a large number of mutexes is being instantiated in a dynamically
allocated memory block that has already been zero-initialized.
(Surely a compiler can be smart enough to avoid redundant zero-initialization
in static initialization. __TBD:__ How to guarantee it in
dynamic memory allocation? [Discussion on GCC
`-flifetime-dse`](https://gcc.gnu.org/legacy-ml/gcc/2016-02/msg00207.html):
"For an object of non-POD class type ... before the constructor
begins execution ... referring to any non-static member or base
class of the object results in undefined behavior."

We may also want to leave room for implementations where
`atomic::wait()` and `atomic::notify_one()` do not trivially map to
operating system primitives and therefore `atomic_mutex` and
`atomic_shared_mutex` could be implemented in a different way, which
might require nontrivial (non-zero) initialization of objects.

# Implementation Experience

An implementation of `atomic_mutex` exists, based on
`atomic<uint32_t>`. An implementation of the `atomic_shared_mutex`
exists, encapsulating `atomic_mutex` and `atomic<uint32_t>`.
It has been tested on GNU/Linux with GCC 11.2.0 and Clang 12 and 13,
as well as with the latest Microsoft Visual Studio on Microsoft Windows.

The implementation also supports C++11 by emulating the C++20
`atomic::wait()` and `atomic::notify_one()` via `futex` system calls
on Linux or OpenBSD.

The prototype implementation [@atomic_sync] is based on a C++11
implementation that is part of [@mariadb_server] 10.6 (using
futex-like operations on Linux, OpenBSD, and Microsoft Windows).

# Future Work

Because `condition_variable` only works with `mutex` and not
`atomic_mutex`, we might want to introduce `atomic_condition_variable`.

Should we define `native_handle`?

---
references:
  - id: atomic_sync
    citation-label: atomic_sync
    title: "Slim mutex and rw-lock using C++ std::atomic"
    URL: https://github.com/dr-m/atomic_sync
  - id: mariadb_server
    citation-label: MariaDB Server
    title: "MariaDB: The open source relational database"
    URL: https://github.com/MariaDB/server/
---
