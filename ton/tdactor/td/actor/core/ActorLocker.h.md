[ChatGPT](https://chat.openai.com/c/862cebfc-1a83-431a-85e6-83a6041fd079)

<details>
  <summary>

    Source Code

  </summary>
  
```cpp

#pragma once

#include "td/actor/core/ActorSignals.h"
#include "td/actor/core/ActorState.h"

#include "td/utils/logging.h"

#include <atomic>

namespace td {
namespace actor {
namespace core {
class ActorLocker {
 public:
  struct Options {
    Options() {
    }
    bool can_execute_paused = false;
    bool is_shared = true;
    SchedulerId scheduler_id;

    Options &with_can_execute_paused(bool new_can_execute_paused) {
      can_execute_paused = new_can_execute_paused;
      return *this;
    }
    Options &with_is_shared(bool new_is_shared) {
      is_shared = new_is_shared;
      return *this;
    }
    Options &with_scheduler_id(SchedulerId id) {
      scheduler_id = id;
      return *this;
    }
  };
  explicit ActorLocker(ActorState *state, Options options = {})
      : state_(state), flags_(state->get_flags_unsafe()), new_flags_{}, options_{options} {
  }
  bool try_lock() {
    CHECK(!own_lock());
    while (!can_try_add_signals()) {
      new_flags_ = flags_;
      new_flags_.set_locked(true);
      new_flags_.clear_signals();
      if (state_->state_.compare_exchange_strong(flags_.raw_ref(), new_flags_.raw(), std::memory_order_acq_rel)) {
        own_lock_ = true;
        return true;
      }
    }
    return false;
  }
  bool try_unlock(ActorState::Flags flags) {
    CHECK(!flags.is_locked());
    CHECK(own_lock());
    // can't unlock with signals set
    //CHECK(!flags.has_signals());

    flags_ = flags;
    // try to unlock
    if (state_->state_.compare_exchange_strong(new_flags_.raw_ref(), flags.raw(), std::memory_order_acq_rel)) {
      own_lock_ = false;
      return true;
    }

    // read all signals
    flags.set_locked(true);
    flags.clear_signals();
    do {
      flags_.add_signals(new_flags_.get_signals());
    } while (!state_->state_.compare_exchange_strong(new_flags_.raw_ref(), flags.raw(), std::memory_order_acq_rel));
    new_flags_ = flags;
    return false;
  }

  bool try_add_signals(ActorSignals signals) {
    CHECK(!own_lock());
    CHECK(can_try_add_signals());
    new_flags_ = flags_;
    new_flags_.add_signals(signals);

    // This is not an optimization.
    // Sometimes it helps sometimes it makes things worse
    // It there are a lot of threads concurrently sending signals to an actor it helps
    // Buf it threre is only one thread, CAS without conficts is much cheaper than full
    // barrier.
    if (false && flags_.raw() == new_flags_.raw()) {
      std::atomic_thread_fence(std::memory_order_seq_cst);
      auto actual_flags = state_->get_flags_unsafe();
      if (actual_flags.raw() == new_flags_.raw()) {
        return true;
      }
    }

    return state_->state_.compare_exchange_strong(flags_.raw_ref(), new_flags_.raw(), std::memory_order_acq_rel);
  }
  bool add_signals(ActorSignals signals) {
    CHECK(!own_lock());
    while (true) {
      if (can_try_add_signals()) {
        if (try_add_signals(signals)) {
          return false;
        }
      } else {
        if (try_lock()) {
          flags_.add_signals(signals);
          return true;
        }
      }
    }
  }
  bool own_lock() const {
    return own_lock_;
  }
  ActorState::Flags flags() const {
    return flags_;
  }
  bool can_execute() const {
    return flags_.is_shared() == options_.is_shared && flags_.get_scheduler_id() == options_.scheduler_id &&
           (options_.can_execute_paused || !flags_.get_signals().has_signal(ActorSignals::Pause));
  }

 private:
  ActorState *state_{nullptr};
  ActorState::Flags flags_;
  ActorState::Flags new_flags_;
  bool own_lock_{false};
  Options options_;

  bool can_try_add_signals() const {
    return flags_.is_locked() || (flags_.is_in_queue() && !can_execute());
  }
};
}  // namespace core
}  // namespace actor
}  // namespace td


  ```
</details>

This C++ code defines the implementation of the `ActorLocker` class, which appears to be part of a larger system related to actors in a concurrent programming context. Let's break down the key components and functionality of this code:

1. **Header Inclusions:**
   - The code includes several headers related to actor functionality, logging, and utility functions.

2. **Namespace:**
   - The code is within the `td::actor::core` namespace.

3. **ActorLocker Class:**
   - The `ActorLocker` class is declared, and it is part of the actor core functionality.
   - It has an inner `Options` struct with several configuration options for the locker.
   - The class contains methods and member variables that are crucial for managing the locking mechanism associated with actor states.

4. **Constructor:**
   - The constructor initializes the `ActorLocker` with an `ActorState` and an optional set of configuration options.

5. **try_lock Method:**
   - The `try_lock` method attempts to acquire a lock on the associated `ActorState` while respecting certain conditions.
   - It uses compare-and-swap operations to perform the locking atomically.

6. **try_unlock Method:**
   - The `try_unlock` method attempts to release a lock on the associated `ActorState`.
   - It checks for certain conditions before unlocking and uses compare-and-swap operations.

7. **try_add_signals Method:**
   - The `try_add_signals` method attempts to add signals to the associated actor atomically.
   - It checks conditions and uses compare-and-swap operations for thread safety.

8. **add_signals Method:**
   - The `add_signals` method is a higher-level method that tries to add signals, and if it fails, it acquires a lock and then adds the signals.

9. **own_lock Method:**
   - The `own_lock` method returns whether the locker currently owns the lock.

10. **flags Method:**
    - The `flags` method returns the current flags associated with the actor state.

11. **can_execute Method:**
    - The `can_execute` method checks if the actor can execute based on certain conditions, including whether it's shared, the scheduler ID, and pause signals.

In summary, this code seems to be a part of an actor framework, providing a mechanism for safely locking and unlocking actor states and managing signals associated with actor execution. The `ActorLocker` class is crucial for coordinating the concurrent execution of actors within the larger system.
