[ChatGPT](https://chat.openai.com/c/89fc7ed5-a0f5-421a-ad75-f5e0822eaa19)

<details>
  <summary>

    Source Code

  </summary>
  
```cpp

#pragma once

#include "td/actor/core/ActorState.h"
#include "td/actor/core/ActorMailbox.h"

#include "td/utils/Heap.h"
#include "td/utils/List.h"
#include "td/utils/Time.h"
#include "td/utils/SharedObjectPool.h"

namespace td {
namespace actor {
namespace core {
class Actor;
class ActorInfo;
using ActorInfoPtr = SharedObjectPool<ActorInfo>::Ptr;
class ActorInfo : private HeapNode, private ListNode {
 public:
  ActorInfo(std::unique_ptr<Actor> actor, ActorState::Flags state_flags, Slice name)
      : actor_(std::move(actor)), name_(name.begin(), name.size()) {
    state_.set_flags_unsafe(state_flags);
    VLOG(actor) << "Create actor [" << name_ << "]";
  }
  ~ActorInfo() {
    VLOG(actor) << "Destroy actor [" << name_ << "]";
    CHECK(!actor_);
  }

  bool is_alive() const {
    return !state_.get_flags_unsafe().is_closed();
  }

  bool has_actor() const {
    return bool(actor_);
  }
  Actor &actor() {
    CHECK(has_actor());
    return *actor_;
  }
  Actor *actor_ptr() const {
    return actor_.get();
  }
  void destroy_actor() {
    actor_.reset();
  }
  ActorState &state() {
    return state_;
  }
  ActorMailbox &mailbox() {
    return mailbox_;
  }
  CSlice get_name() const {
    return name_;
  }

  HeapNode *as_heap_node() {
    return this;
  }
  static ActorInfo *from_heap_node(HeapNode *node) {
    return static_cast<ActorInfo *>(node);
  }

  Timestamp get_alarm_timestamp() const {
    return Timestamp::at(alarm_timestamp_at_.load(std::memory_order_relaxed));
  }
  void set_alarm_timestamp(Timestamp timestamp) {
    alarm_timestamp_at_.store(timestamp.at(), std::memory_order_relaxed);
  }

  void pin(ActorInfoPtr ptr) {
    CHECK(pin_.empty());
    CHECK(&*ptr == this);
    pin_ = std::move(ptr);
  }
  ActorInfoPtr unpin() {
    CHECK(!pin_.empty());
    return std::move(pin_);
  }

 private:
  std::unique_ptr<Actor> actor_;
  ActorState state_;
  ActorMailbox mailbox_;
  std::string name_;
  std::atomic<double> alarm_timestamp_at_{0};

  ActorInfoPtr pin_;
};

}  // namespace core
}  // namespace actor
}  // namespace td


  ```
</details>

This code appears to be part of the implementation of an actor framework, likely related to the mentioned blockchain code known as TON (The Open Network). Let's break down the key components and functionalities:

1. **Header Inclusions:**
   - The code includes various headers from the `td` and `td/utils` namespaces, which might be part of a larger framework or library.
   - Headers like `ActorState.h`, `ActorMailbox.h`, `Heap.h`, `List.h`, `Time.h`, and `SharedObjectPool.h` suggest involvement in an actor-based system with features related to memory management and time tracking.

2. **Namespace and Class Declarations:**
   - The code is within the `td::actor::core` namespace.
   - There's a declaration for the `ActorInfo` class and an associated `ActorInfoPtr` type, which appears to be a smart pointer to `ActorInfo` managed by a shared object pool.

3. **Constructor and Destructor:**
   - The constructor takes a unique pointer to an `Actor`, `ActorState::Flags`, and a `Slice` (presumably a string).
   - It logs the creation of an actor.
   - The destructor logs the destruction of an actor and ensures that the actor is no longer valid.

4. **Public Methods:**
   - `is_alive()`: Checks if the actor is alive (not closed).
   - `has_actor()`: Checks if the actor exists.
   - `actor()`: Returns a reference to the actor.
   - `actor_ptr()`: Returns a pointer to the actor.
   - `destroy_actor()`: Resets the actor pointer, effectively destroying the actor.
   - `state()`: Returns a reference to the actor's state.
   - `mailbox()`: Returns a reference to the actor's mailbox.
   - `get_name()`: Returns the name of the actor.
   - `as_heap_node()`: Returns a pointer to the object as a `HeapNode`.
   - `from_heap_node()`: Converts a `HeapNode` pointer to an `ActorInfo` pointer.
   - `get_alarm_timestamp()`, `set_alarm_timestamp()`: Get and set the alarm timestamp of the actor.

5. **Private Members:**
   - `actor_`: A unique pointer to the actor.
   - `state_`: An instance of `ActorState` representing the actor's state.
   - `mailbox_`: An instance of `ActorMailbox` representing the actor's mailbox.
   - `name_`: A string representing the name of the actor.
   - `alarm_timestamp_at_`: An atomic variable holding the alarm timestamp.
   - `pin_`: An `ActorInfoPtr` for pinning the actor.

6. **Pin and Unpin Methods:**
   - `pin()`: Pins the actor by storing an `ActorInfoPtr`.
   - `unpin()`: Unpins the actor by returning the stored `ActorInfoPtr`.

In summary, this code defines the `ActorInfo` class, which encapsulates information about an actor in an actor-based system, including the actor's state, mailbox, name, and other related functionalities. It seems to be part of a larger framework designed for building complex systems, possibly within the context of a blockchain implementation.
