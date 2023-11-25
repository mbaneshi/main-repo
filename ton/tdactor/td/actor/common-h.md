[ChatGPT](https://chat.openai.com/c/0f30cd00-2463-406a-a12e-5d2614131c86)

<details>
  <summary>

    Source Code

  </summary>
  
```cpp

#pragma once
#include "td/actor/core/Actor.h"
#include "td/actor/core/ActorSignals.h"
#include "td/actor/core/SchedulerId.h"
#include "td/actor/core/SchedulerContext.h"
#include "td/actor/core/Scheduler.h"

#include "td/actor/PromiseFuture.h"

#include "td/utils/Timer.h"

namespace td {
namespace actor {
using core::ActorOptions;

// Replacement for core::ActorSignals. Easier to use and do not allow internal signals
class ActorSignals {
 public:
  static ActorSignals pause() {
    return ActorSignals(core::ActorSignals::one(core::ActorSignals::Pause));
  }
  static ActorSignals kill() {
    return ActorSignals(core::ActorSignals::one(core::ActorSignals::Kill));
  }
  static ActorSignals wakeup() {
    return ActorSignals(core::ActorSignals::one(core::ActorSignals::Wakeup));
  }
  friend ActorSignals operator|(ActorSignals a, ActorSignals b) {
    a.raw_.add_signals(b.raw_);
    return a;
  }
  ActorSignals() = default;

  core::ActorSignals raw() const {
    return raw_;
  }

 private:
  ActorSignals(core::ActorSignals raw) : raw_(raw) {
  }
  core::ActorSignals raw_;
};

// TODO: proper interface
using core::Actor;
using core::SchedulerContext;
using core::SchedulerId;
using core::set_debug;

struct Debug {
 public:
  Debug() = default;
  Debug(std::shared_ptr<core::SchedulerGroupInfo> group_info) : group_info_(std::move(group_info)) {
  }
  template <class F>
  void for_each(F &&f) {
    for (auto &scheduler : group_info_->schedulers) {
      f(scheduler.io_worker->debug);
      for (auto &cpu : scheduler.cpu_workers) {
        f(cpu->debug);
      }
    }
  }

  void dump() {
    for_each([](core::Debug &debug) {
      core::DebugInfo info;
      debug.read(info);
      if (info.is_active) {
        LOG(ERROR) << info.name << " " << td::format::as_time(Time::now() - info.start_at);
      }
    });
  }

 private:
  std::shared_ptr<core::SchedulerGroupInfo> group_info_;
};

class Scheduler {
 public:
  struct NodeInfo {
    NodeInfo(size_t cpu_threads) : cpu_threads_(cpu_threads) {
    }
    NodeInfo(size_t cpu_threads, size_t io_threads) : cpu_threads_(cpu_threads), io_threads_(io_threads) {
    }
    size_t cpu_threads_;
    size_t io_threads_{1};
  };

  enum Mode { Running, Paused };
  Scheduler(std::vector<NodeInfo> infos, bool skip_timeouts = false, Mode mode = Paused)
      : infos_(std::move(infos)), skip_timeouts_(skip_timeouts) {
    init();
    if (mode == Running) {
      start();
    }
  }

  ~Scheduler() {
    stop();
  }
  Scheduler(const Scheduler &) = delete;
  Scheduler(Scheduler &&) = delete;
  Scheduler &operator=(const Scheduler &) = delete;
  Scheduler &operator=(Scheduler &&) = delete;

  void start() {
    if (is_started_) {
      return;
    }
    is_started_ = true;
    for (size_t it = 0; it < schedulers_.size(); it++) {
      auto &scheduler = schedulers_[it];
      scheduler->start();
      if (it != 0) {
        auto thread = td::thread([&] {
          while (scheduler->run(10)) {
          }
        });
        thread.set_name(PSLICE() << "#" << it << ":io");
        thread.detach();
      }
    }
  }

  Debug get_debug() {
    return Debug{group_info_};
  }

  bool run() {
    start();
    while (schedulers_[0]->run(10)) {
    }
    return false;
  }

  bool run(double timeout) {
    start();
    return schedulers_[0]->run(timeout);
  }

  template <class F>
  void run_in_context(F &&f) {
    schedulers_[0]->run_in_context(std::forward<F>(f));
  }

  template <class F>
  void run_in_context_external(F &&f) {
    schedulers_[0]->run_in_context_external(std::forward<F>(f));
  }

  void stop() {
    if (!group_info_) {
      return;
    }
    if (!is_started_) {
      start();
    }
    schedulers_[0]->stop();
    run();
    core::Scheduler::close_scheduler_group(*group_info_);
    group_info_.reset();
  }

 private:
  std::vector<NodeInfo> infos_{td::thread::hardware_concurrency()};
  std::shared_ptr<core::SchedulerGroupInfo> group_info_;
  std::vector<td::unique_ptr<core::Scheduler>> schedulers_;
  bool is_started_{false};
  bool skip_timeouts_{false};

  void init() {
    CHECK(infos_.size() < 256);
    CHECK(!group_info_);
    group_info_ = std::make_shared<core::SchedulerGroupInfo>(infos_.size());
    td::uint8 id = 0;
    for (const auto &info : infos_) {
      schedulers_.emplace_back(
          td::make_unique<core::Scheduler>(group_info_, core::SchedulerId{id}, info.cpu_threads_, skip_timeouts_));
      id++;
    }
  }
};

// Some helper functions. Not part of public interface and not part
// of namespace core
namespace detail {
template <class LambdaT>
class ActorMessageLambda : public core::ActorMessageImpl {
 public:
  template <class FromLambdaT>
  explicit ActorMessageLambda(FromLambdaT &&lambda) : lambda_(std::forward<FromLambdaT>(lambda)) {
  }
  void run() override {
    //delivery_warning_.reset();
    lambda_();
  }

 private:
  LambdaT lambda_;
  //PerfWarningTimer delivery_warning_{"Long message delivery", 2};
};

class ActorMessageCreator {
 public:
  template <class F>
  static auto lambda(F &&f) {
    return core::ActorMessage(std::make_unique<ActorMessageLambda<std::decay_t<F>>>(std::forward<F>(f)));
  }

  static auto hangup() {
    return core::ActorMessage(std::make_unique<core::ActorMessageHangup>());
  }

  static auto hangup_shared() {
    return core::ActorMessage(std::make_unique<core::ActorMessageHangupShared>());
  }

  // Use faster allocation?
};
struct ActorRef {
  ActorRef(core::ActorInfo &actor_info, uint64 link_token = core::EmptyLinkToken)
      : actor_info(actor_info), link_token(link_token) {
  }

  core::ActorInfo &actor_info;
  uint64 link_token;
};

template <class T>
T &current_actor() {
  return static_cast<T &>(core::ActorExecuteContext::get()->actor());
}

inline void send_message(core::ActorInfo &actor_info, core::ActorMessage message) {
  auto scheduler_context_ptr = core::SchedulerContext::get();
  if (scheduler_context_ptr == nullptr) {
    //LOG(ERROR) << "send to actor is silently ignored";
    return;
  }
  auto &scheduler_context = *scheduler_context_ptr;
  core::ActorExecutor executor(actor_info, scheduler_context,
                               core::ActorExecutor::Options().with_has_poll(scheduler_context.has_poll()));
  executor.send(std::move(message));
}

inline void send_message(ActorRef actor_ref, core::ActorMessage message) {
  message.set_link_token(actor_ref.link_token);
  send_message(actor_ref.actor_info, std::move(message));
}

inline void send_message_later(core::ActorInfo &actor_info, core::ActorMessage message) {
  auto scheduler_context_ptr = core::SchedulerContext::get();
  if (scheduler_context_ptr == nullptr) {
    //LOG(ERROR) << "send to actor is silently ignored";
    return;
  }
  auto &scheduler_context = *scheduler_context_ptr;
  core::ActorExecutor executor(actor_info, scheduler_context,
                               core::ActorExecutor::Options().with_has_poll(scheduler_context.has_poll()));
  message.set_big();
  executor.send(std::move(message));
}

inline void send_message_later(ActorRef actor_ref, core::ActorMessage message) {
  message.set_link_token(actor_ref.link_token);
  send_message_later(actor_ref.actor_info, std::move(message));
}

template <class ExecuteF, class ToMessageF>
void send_immediate(ActorRef actor_ref, ExecuteF &&execute, ToMessageF &&to_message) {
  auto scheduler_context_ptr = core::SchedulerContext::get();
  if (scheduler_context_ptr == nullptr) {
    //LOG(ERROR) << "send to actor is silently ignored";
    return;
  }
  auto &scheduler_context = *scheduler_context_ptr;
  core::ActorExecutor executor(actor_ref.actor_info, scheduler_context,
                               core::ActorExecutor::Options().with_has_poll(scheduler_context.has_poll()));
  if (executor.can_send_immediate()) {
    return executor.send_immediate(execute, actor_ref.link_token);
  }
  auto message = to_message();
  message.set_link_token(actor_ref.link_token);
  executor.send(std::move(message));
}

template <class F>
void send_lambda(ActorRef actor_ref, F &&lambda) {
  send_immediate(actor_ref, lambda, [&lambda]() mutable { return ActorMessageCreator::lambda(std::move(lambda)); });
}
template <class F>
void send_lambda_later(ActorRef actor_ref, F &&lambda) {
  send_message_later(actor_ref, ActorMessageCreator::lambda(std::move(lambda)));
}

template <class ClosureT>
void send_closure_impl(ActorRef actor_ref, ClosureT &&closure) {
  using ActorType = typename ClosureT::ActorType;
  send_immediate(
      actor_ref, [&closure]() mutable { closure.run(&current_actor<ActorType>()); },
      [&closure]() mutable {
        return ActorMessageCreator::lambda(
            [closure = to_delayed_closure(std::move(closure))]() mutable { closure.run(&current_actor<ActorType>()); });
      });
}

template <class... ArgsT>
void send_closure(ActorRef actor_ref, ArgsT &&... args) {
  send_closure_impl(actor_ref, create_immediate_closure(std::forward<ArgsT>(args)...));
}

template <class ClosureT>
void send_closure_later_impl(ActorRef actor_ref, ClosureT &&closure) {
  using ActorType = typename ClosureT::ActorType;
  send_message_later(actor_ref,
                     ActorMessageCreator::lambda([closure = to_delayed_closure(std::move(closure))]() mutable {
                       closure.run(&current_actor<ActorType>());
                     }));
}

template <class ClosureT, class PromiseT>
void send_closure_with_promise(ActorRef actor_ref, ClosureT &&closure, PromiseT &&promise) {
  using ActorType = typename ClosureT::ActorType;
  using ResultType = decltype(closure.run(&current_actor<ActorType>()));
  auto &&promise_i = promise_interface<ResultType>(std::forward<PromiseT>(promise));
  send_immediate(
      actor_ref, [&closure, &promise = promise_i]() mutable { promise(closure.run(&current_actor<ActorType>())); },
      [&closure, &promise = promise_i]() mutable {
        return ActorMessageCreator::lambda(
            [closure = to_delayed_closure(std::move(closure)), promise = std::move(promise)]() mutable {
              promise(closure.run(&current_actor<ActorType>()));
            });
      });
}

template <class ClosureT, class PromiseT>
void send_closure_with_promise_later(ActorRef actor_ref, ClosureT &&closure, PromiseT &&promise) {
  using ActorType = typename ClosureT::ActorType;
  using ResultType = decltype(closure.run(&current_actor<ActorType>()));
  send_message_later(
      actor_ref,
      ActorMessageCreator::lambda([closure = to_delayed_closure(std::move(closure)),
                                   promise = promise_interface<ResultType>(std::forward<PromiseT>(promise))]() mutable {
        promise(closure.run(&current_actor<ActorType>()));
      }));
}

template <class... ArgsT>
void send_closure_later(ActorRef actor_ref, ArgsT &&... args) {
  send_closure_later_impl(actor_ref, create_delayed_closure(std::forward<ArgsT>(args)...));
}

inline void send_signals(ActorRef actor_ref, ActorSignals signals) {
  auto scheduler_context_ptr = core::SchedulerContext::get();
  if (scheduler_context_ptr == nullptr) {
    //LOG(ERROR) << "send to actor is silently ignored";
    return;
  }
  auto &scheduler_context = *scheduler_context_ptr;
  core::ActorExecutor executor(
      actor_ref.actor_info, scheduler_context,
      core::ActorExecutor::Options().with_has_poll(scheduler_context.has_poll()).with_signals(signals.raw()));
}

inline void send_signals_later(ActorRef actor_ref, ActorSignals signals) {
  auto scheduler_context_ptr = core::SchedulerContext::get();
  if (scheduler_context_ptr == nullptr) {
    //LOG(ERROR) << "send to actor is silently ignored";
    return;
  }
  auto &scheduler_context = *scheduler_context_ptr;
  core::ActorExecutor executor(actor_ref.actor_info, scheduler_context,
                               core::ActorExecutor::Options()
                                   .with_has_poll(scheduler_context.has_poll())
                                   .with_signals((signals | ActorSignals::pause()).raw()));
}

inline void register_actor_info_ptr(core::ActorInfoPtr actor_info_ptr) {
  auto state = actor_info_ptr->state().get_flags_unsafe();
  core::SchedulerContext::get()->add_to_queue(std::move(actor_info_ptr), state.get_scheduler_id(), !state.is_shared());
}

template <class T, class... ArgsT>
core::ActorInfoPtr create_actor(core::ActorOptions &options, ArgsT &&... args) noexcept {
  auto *scheduler_context = core::SchedulerContext::get();
  if (!options.has_scheduler()) {
    options.on_scheduler(scheduler_context->get_scheduler_id());
  }
  auto res =
      scheduler_context->get_actor_info_creator().create(std::make_unique<T>(std::forward<ArgsT>(args)...), options);
  register_actor_info_ptr(res);
  return res;
}
}  // namespace detail
}  // namespace actor
}  // namespace td


  ```
</details>

The provided code is a part of the TDLib (Telegram Database Library) for C++. It seems to be a set of classes and functions related to actor-based concurrency and scheduling in the context of Telegram's messaging system.

Here's a brief summary of the code:

1. **Namespace and Includes:**
   - The code is in the `td::actor` namespace.
   - It includes several headers related to actors, schedulers, promises, and utility functions.

2. **ActorSignals Class:**
   - Defines a replacement for `core::ActorSignals`, providing easier methods for creating signals (pause, kill, wakeup).
   - Overloaded `|` operator is used for combining signals.

3. **Scheduler Class:**
   - Represents a scheduler that manages a group of actors.
   - It has two modes: Running and Paused.
   - It can start, stop, and run actors.

4. **Debug Class:**
   - Provides a way to collect and dump debug information about schedulers and their workers.

5. **Scheduler-related Classes:**
   - `NodeInfo` stores information about the number of CPU and I/O threads.
   - `SchedulerId` is an identifier for a scheduler.
   - `SchedulerContext` manages the context for a scheduler.

6. **ActorMessageLambda Class:**
   - A helper class for creating actor messages with lambda functions.

7. **ActorRef Struct:**
   - Represents a reference to an actor, including an actor's information and a link token.

8. **Detail Namespace:**
   - Contains various helper functions for sending messages, closures, and signals to actors.

9. **Utility Functions:**
   - `send_message`: Sends a message to an actor.
   - `send_message_later`: Sends a message to an actor with a delay.
   - `send_closure`: Sends a closure to an actor for immediate execution.
   - `send_closure_with_promise`: Sends a closure with a promise to an actor for immediate execution.
   - `send_signals`: Sends signals to an actor.
   - `create_actor`: Creates an actor of a specified type with options.

10. **Miscellaneous:**
   - There are some commented-out lines, indicating potential debugging or logging statements.

This code seems to be a part of a larger framework for handling concurrent and asynchronous tasks using the actor model, as it's often used in messaging systems to handle multiple concurrent tasks efficiently. If you have specific questions or if there's a particular aspect you'd like more detail on, feel free to ask!