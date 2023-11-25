[ChatGPT](https://chat.openai.com/c/ce5200ab-5af4-4906-b573-82496f65acb0)

<details>
  <summary>

    Source Code

  </summary>
  
```cpp

#pragma once

#include "td/utils/common.h"
#include "td/utils/bits.h"
#include "td/utils/format.h"

namespace td {
namespace actor {
namespace core {
class ActorSignals {
 public:
  ActorSignals() = default;
  uint32 raw() const {
    return raw_;
  }
  static ActorSignals create_raw(uint32 raw) {
    return ActorSignals(raw);
  }
  bool empty() const {
    return raw_ == 0;
  }
  bool has_signal(uint32 signal) const {
    return (raw_ & (1u << signal)) != 0;
  }
  void add_signal(uint32 signal) {
    raw_ |= (1u << signal);
  }
  void add_signals(ActorSignals signals) {
    raw_ |= signals.raw();
  }
  void clear_signal(uint32 signal) {
    raw_ &= ~(1u << signal);
  }
  uint32 first_signal() {
    if (!raw_) {
      return 0;
    }
    return td::count_trailing_zeroes_non_zero32(raw_);
  }
  friend StringBuilder &operator<<(StringBuilder &sb, ActorSignals signals) {
    sb << "S{";
    bool was = false;
    auto add_signal = [&](int signal, auto name) {
      if (signals.has_signal(signal)) {
        if (was) {
          sb << ",";
        } else {
          was = true;
        }
        sb << name;
      }
    };
    add_signal(Wakeup, "Wakeup");
    add_signal(Alarm, "Alarm");
    add_signal(Kill, "Kill");
    add_signal(Io, "Io");
    add_signal(Cpu, "Cpu");
    add_signal(StartUp, "StartUp");
    add_signal(Pop, "Pop");
    add_signal(Message, "Message");
    add_signal(Pause, "Pause");
    sb << "}";
    return sb;
  }
  enum Signal : uint32 {
    // Signals in order of priority
    Pause = 1,
    Kill = 2,  // immediate kill
    StartUp = 3,
    Wakeup = 4,
    Alarm = 5,
    Io = 6,   // move to io thread
    Cpu = 7,  // move to cpu thread
    // Two signals for mpmc queue logic
    //
    // PopSignal is set after actor is popped from queue
    // When processed it should set InQueue and Pause flags to false.
    //
    // MessagesSignal is set after new messages was added to actor
    // If owner of actor wish to delay message handling, she should set InQueue flag to true and
    // add actor into mpmc queue.
    Pop = 8,      // got popped from queue
    Message = 9,  // got new message
  };

  static ActorSignals one(uint32 signal) {
    ActorSignals res;
    res.add_signal(signal);
    return res;
  }

 private:
  uint32 raw_{0};
  friend class ActorState;
  explicit ActorSignals(uint32 raw) : raw_(raw) {
  }
};
}  // namespace core
}  // namespace actor
}  // namespace td


  ```
</details>

This C++ code defines the `ActorSignals` class, which seems to be a part of a larger system related to actors in a concurrent programming context. Here's a summary of the key aspects of this code:

1. **Class Definition:**
   - The `ActorSignals` class is declared within the `core` namespace under the `actor` namespace of the `td` namespace.
   - It contains methods and data members to manipulate and query signals associated with actors.

2. **Signal Enumeration:**
   - The class includes an enumeration named `Signal` that defines various signals with associated priority levels.
   - Signals include `Pause`, `Kill`, `StartUp`, `Wakeup`, `Alarm`, `Io`, `Cpu`, `Pop`, and `Message`.
   - These signals are associated with different events or actions related to actor behavior.

3. **Methods:**
   - The class provides methods to manipulate the signals, such as `add_signal`, `clear_signal`, `add_signals`, and `has_signal`.
   - The `raw()` method returns the raw representation of signals.
   - The `first_signal()` method returns the priority of the first signal that is set.

4. **StringBuilder Overload:**
   - The class overloads the `<<` operator for `StringBuilder`. This allows instances of `ActorSignals` to be easily converted to a string representation.

5. **Static Methods:**
   - There are static methods like `create_raw` and `one` to create instances of `ActorSignals` with specific signal configurations.

6. **Friend Class:**
   - The `ActorSignals` class declares `ActorState` as a friend class, suggesting a close relationship with another class, possibly for managing actor state.

7. **Namespace Dependencies:**
   - The class relies on several utilities from the `td` namespace, including `common.h`, `bits.h`, and `format.h`.

8. **Signal Output:**
   - The `<<` operator overload outputs the set signals in a human-readable format.

This code appears to be a component of a larger system for managing actor behavior, with signals indicating different events or actions that can be associated with actors in a concurrent environment. Understanding its full functionality would likely require examining other parts of the codebase, particularly classes like `ActorState` and their interactions.
