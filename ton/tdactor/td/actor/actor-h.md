[ChatGPT](https://chat.openai.com/c/626f4a13-b6dc-4b6a-b01a-887408ec3bce)
<details>
  <summary>
    Source Code
  </summary>

  
  ```cpp

#pragma once
#include "td/actor/common.h"
#include "td/actor/ActorId.h"
#include "td/actor/ActorOwn.h"
#include "td/actor/ActorShared.h"

namespace td {
namespace actor {

template <class T, class... ArgsT>
TD_WARN_UNUSED_RESULT ActorOwn<T> create_actor(ActorOptions options, ArgsT &&... args) {
  return ActorOwn<T>(ActorId<T>::create(options, std::forward<ArgsT>(args)...));
}

template <class T, class... ArgsT>
TD_WARN_UNUSED_RESULT ActorOwn<T> create_actor(Slice name, ArgsT &&... args) {
  return ActorOwn<T>(ActorId<T>::create(ActorOptions().with_name(name), std::forward<ArgsT>(args)...));
}

#define SEND_CLOSURE_LATER 1
#ifndef SEND_CLOSURE_LATER

template <class ActorIdT, class FunctionT, class... ArgsT, class FunctionClassT = member_function_class_t<FunctionT>,
          size_t argument_count = member_function_argument_count<FunctionT>(),
          std::enable_if_t<argument_count == sizeof...(ArgsT), bool> with_promise = false>
void send_closure(ActorIdT &&actor_id, FunctionT function, ArgsT &&... args) {
  using ActorT = typename std::decay_t<ActorIdT>::ActorT;
  static_assert(std::is_base_of<FunctionClassT, ActorT>::value, "unsafe send_closure");

  ActorIdT id = std::forward<ActorIdT>(actor_id);
  detail::send_closure(id.as_actor_ref(), function, std::forward<ArgsT>(args)...);
}

template <class ActorIdT, class FunctionT, class... ArgsT, class FunctionClassT = member_function_class_t<FunctionT>,
          size_t argument_count = member_function_argument_count<FunctionT>(),
          std::enable_if_t<argument_count != sizeof...(ArgsT), bool> with_promise = true>
void send_closure(ActorIdT &&actor_id, FunctionT function, ArgsT &&... args) {
  using ActorT = typename std::decay_t<ActorIdT>::ActorT;
  static_assert(std::is_base_of<FunctionClassT, ActorT>::value, "unsafe send_closure");

  ActorIdT id = std::forward<ActorIdT>(actor_id);
  detail::send_closure_with_promise(id.as_actor_ref(),
                                    call_n_arguments<argument_count>(
                                        [&function](auto &&... nargs) {
                                          return create_immediate_closure(function,
                                                                          std::forward<decltype(nargs)>(nargs)...);
                                        },
                                        std::forward<ArgsT>(args)...),
                                    get_last_argument(std::forward<ArgsT>(args)...));
}

#else

template <class ActorIdT, class FunctionT, class... ArgsT, class FunctionClassT = member_function_class_t<FunctionT>,
          size_t argument_count = member_function_argument_count<FunctionT>(),
          std::enable_if_t<argument_count == sizeof...(ArgsT), bool> with_promise = false>
void send_closure(ActorIdT &&actor_id, FunctionT function, ArgsT &&... args) {
  using ActorT = typename std::decay_t<ActorIdT>::ActorT;
  static_assert(std::is_base_of<FunctionClassT, ActorT>::value, "unsafe send_closure");

  ActorIdT id = std::forward<ActorIdT>(actor_id);
  detail::send_closure_later(id.as_actor_ref(), function, std::forward<ArgsT>(args)...);
}

template <class ActorIdT, class FunctionT, class... ArgsT, class FunctionClassT = member_function_class_t<FunctionT>,
          size_t argument_count = member_function_argument_count<FunctionT>(),
          std::enable_if_t<argument_count != sizeof...(ArgsT), bool> with_promise = true>
void send_closure(ActorIdT &&actor_id, FunctionT function, ArgsT &&... args) {
  using ActorT = typename std::decay_t<ActorIdT>::ActorT;
  static_assert(std::is_base_of<FunctionClassT, ActorT>::value, "unsafe send_closure");

  ActorIdT id = std::forward<ActorIdT>(actor_id);
  detail::send_closure_with_promise_later(id.as_actor_ref(),
                                          call_n_arguments<argument_count>(
                                              [&function](auto &&... nargs) {
                                                return create_delayed_closure(function,
                                                                              std::forward<decltype(nargs)>(nargs)...);
                                              },
                                              std::forward<ArgsT>(args)...),
                                          get_last_argument(std::forward<ArgsT>(args)...));
}

#endif

template <class ActorIdT, class FunctionT, class... ArgsT, class FunctionClassT = member_function_class_t<FunctionT>,
          size_t argument_count = member_function_argument_count<FunctionT>(),
          std::enable_if_t<argument_count == sizeof...(ArgsT), bool> with_promise = false>
auto future_send_closure(ActorIdT &&actor_id, FunctionT function, ArgsT &&... args) {
  using R = ::td::detail::get_ret_t<std::decay_t<FunctionT>>;
  auto pf = make_promise_future<R>();
  send_closure(std::forward<ActorIdT>(actor_id), std::move(function), std::forward<ArgsT>(args)...,
               std::move(pf.first));
  return std::move(pf.second);
}

template <class R, class ActorIdT, class FunctionT, class... ArgsT,
          class FunctionClassT = member_function_class_t<FunctionT>,
          size_t argument_count = member_function_argument_count<FunctionT>(),
          std::enable_if_t<argument_count != sizeof...(ArgsT), bool> with_promise = true>
Future<R> future_send_closure(ActorIdT &&actor_id, FunctionT function, ArgsT &&... args) {
  auto pf = make_promise_future<R>();
  send_closure(std::forward<ActorIdT>(actor_id), std::move(function), std::forward<ArgsT>(args)...,
               std::move(pf.first));
  return std::move(pf.second);
}

template <typename ActorIdT, typename FunctionT, typename... ArgsT>
bool send_closure_bool(ActorIdT &&actor_id, FunctionT function, ArgsT &&... args) {
  send_closure(std::forward<ActorIdT>(actor_id), function, std::forward<ArgsT>(args)...);
  return true;
}

template <class ActorIdT, class FunctionT, class... ArgsT, class FunctionClassT = member_function_class_t<FunctionT>,
          size_t argument_count = member_function_argument_count<FunctionT>(),
          std::enable_if_t<argument_count == sizeof...(ArgsT), bool> with_promise = false>
void send_closure_later(ActorIdT &&actor_id, FunctionT function, ArgsT &&... args) {
  using ActorT = typename std::decay_t<ActorIdT>::ActorT;
  static_assert(std::is_base_of<FunctionClassT, ActorT>::value, "unsafe send_closure");

  ActorIdT id = std::forward<ActorIdT>(actor_id);
  detail::send_closure_later(id.as_actor_ref(), function, std::forward<ArgsT>(args)...);
}

template <class ActorIdT, class FunctionT, class... ArgsT, class FunctionClassT = member_function_class_t<FunctionT>,
          size_t argument_count = member_function_argument_count<FunctionT>(),
          std::enable_if_t<argument_count != sizeof...(ArgsT), bool> with_promise = true>
void send_closure_later(ActorIdT &&actor_id, FunctionT function, ArgsT &&... args) {
  using ActorT = typename std::decay_t<ActorIdT>::ActorT;
  static_assert(std::is_base_of<FunctionClassT, ActorT>::value, "unsafe send_closure");

  ActorIdT id = std::forward<ActorIdT>(actor_id);
  detail::send_closure_with_promise_later(id.as_actor_ref(),
                                          call_n_arguments<argument_count>(
                                              [&function](auto &&... nargs) {
                                                return create_delayed_closure(function,
                                                                              std::forward<decltype(nargs)>(nargs)...);
                                              },
                                              std::forward<ArgsT>(args)...),
                                          get_last_argument(std::forward<ArgsT>(args)...));
}

template <typename ActorIdT, typename FunctionT, typename... ArgsT>
bool send_closure_later_bool(ActorIdT &&actor_id, FunctionT function, ArgsT &&... args) {
  send_closure_later(std::forward<ActorIdT>(actor_id), function, std::forward<ArgsT>(args)...);
  return true;
}

template <class ActorIdT, class... ArgsT>
void send_lambda(ActorIdT &&actor_id, ArgsT &&... args) {
  ActorIdT id = std::forward<ActorIdT>(actor_id);
  detail::send_lambda(id.as_actor_ref(), std::forward<ArgsT>(args)...);
}
template <class ActorIdT, class... ArgsT>
void send_lambda_later(ActorIdT &&actor_id, ArgsT &&... args) {
  ActorIdT id = std::forward<ActorIdT>(actor_id);
  detail::send_lambda_later(id.as_actor_ref(), std::forward<ArgsT>(args)...);
}
template <class ActorIdT>
void send_signals(ActorIdT &&actor_id, ActorSignals signals) {
  ActorIdT id = std::forward<ActorIdT>(actor_id);
  detail::send_signals(id.as_actor_ref(), signals);
}
template <class ActorIdT>
void send_signals_later(ActorIdT &&actor_id, ActorSignals signals) {
  ActorIdT id = std::forward<ActorIdT>(actor_id);
  detail::send_signals_later(id.as_actor_ref(), signals);
}
}  // namespace actor

class SendClosure {
 public:
  template <class... ArgsT>
  void operator()(ArgsT &&... args) const {
    td::actor::send_closure(std::forward<ArgsT>(args)...);
  }
};

template <class T>
template <class... ArgsT>
auto Promise<T>::send_closure(ArgsT &&... args) {
  return [promise = std::move(*this), t = std::make_tuple(std::forward<ArgsT>(args)...)](auto &&r_res) mutable {
    TRY_RESULT_PROMISE(promise, res, std::move(r_res));
    td::call_tuple(SendClosure(), std::tuple_cat(std::move(t), std::make_tuple(std::move(res), std::move(promise))));
  };
}

template <class... ArgsT>
auto promise_send_closure(ArgsT &&... args) {
  return [t = std::make_tuple(std::forward<ArgsT>(args)...)](auto &&res) mutable {
    td::call_tuple(SendClosure(), std::tuple_cat(std::move(t), std::make_tuple(std::move(res))));
  };
}

}  // namespace td

```

</details>

This C++ code appears to be part of the TON (The Open Network) blockchain codebase, specifically related to the actor framework used in TON. Let's break down the key components and functionality of this code:

1. **Actor Creation:**
   - The `create_actor` function is a template function used to create actors in the TON actor framework.
   - It takes actor options and a variable number of arguments to construct the actor.

2. **SEND_CLOSURE_LATER:**
   - This macro determines whether to use immediate or delayed closure when sending messages to actors.
   - Immediate closure means the closure is executed immediately, while delayed closure is executed later.

3. **send_closure Function:**
   - The `send_closure` function is a template function for sending closures to actors.
   - It is overloaded based on whether immediate or delayed closure is used.
   - It takes an actor ID, a function, and arguments for the function.
   - The closures are sent to actors using the `detail::send_closure` or `detail::send_closure_later` functions.

4. **future_send_closure Function:**
   - The `future_send_closure` function is a template function for sending closures to actors and obtaining a future result.
   - It returns a future representing the result of the closure execution.

5. **send_closure_bool and send_closure_later_bool:**
   - These functions are specialized versions of `send_closure` and `send_closure_later` that return a boolean indicating success.

6. **send_lambda and send_lambda_later:**
   - These functions are for sending lambda functions to actors, either immediately or later.

7. **send_signals and send_signals_later:**
   - These functions are for sending signals to actors, either immediately or later.

8. **SendClosure Class:**
   - This class is a functor used for sending closures.
   - It wraps the `td::actor::send_closure` function.

9. **Promise Class:**
   - This class seems to be related to promises used in the actor framework.
   - The `send_closure` method generates a closure that sends a tuple of arguments to the actor using `SendClosure`.

10. **promise_send_closure Function:**
   - This function generates a closure for sending a tuple of arguments to an actor using `SendClosure`.

Overall, this code is a part of the actor framework in the TON blockchain codebase, providing functionality for creating actors, sending closures, handling promises, and managing different closure sending modes (immediate or delayed).
