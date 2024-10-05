---
title: Building state machines with std::visit
date: 2024-10-05
categories: [Software Engineering]
tags: [cpp, c++]
comments: false
---

## Introduction

For my first post I would to explore an implementation for state machines that utilizes [std::visit](https://en.cppreference.com/w/cpp/utility/variant/visit) and [std::variant](https://en.cppreference.com/w/cpp/utility/variant). `std::variant` is a type-safe union that the C++ standard library introduced with C++17. `std::visit` utilizes `std::variant` to apply a visitor (lambda, function object, a callable in general) to a set of variants.

## Alarm state machine

Let's take a practical example. We want to implement a (simplified) state machine for a home alarm system. We will start by defining our states and events. We need to define a new type for all of the possible states and events.

*Note:* For simplicity we will only use 3 states (Disarm, Arm, Alarm) and 2 events (Authentication, Door intrusion detection). A complete alarm state machine would have extra states (eg. an intermediate state between ARM and ALARM with a timer to allow some time the user to disarm the alarm when entering home) and events (eg. events from motion detection sensor).

```cpp
namespace states {
    struct disarm{};
    struct arm{};
    struct alarm{};
};

namespace events {
    struct authenticate{};
    struct door{};
};
```

Then, we define a `std::variant` for the states and another one for the events.

```cpp
using State = std::variant<disarm, arm, alarm>;
using Event = std::variant<authenticate, door>;
```

Now that we have our data types layed out, we are ready to define our state transition matrix by implementing a struct called `transitions` and overloading the call operator based on 2 arguments, a pair of state and event. The state defines the current state of the state machine and event is the most recent event that was captured by our system. There are cases where an event might not trigger a state transition (eg. door sensor trigger while on DISARM) which seems like a good case for using `std::optional`. 

We return the new `State` when there is a valid transition and `std::nullopt` when there is no transition happening. Inside those functions you will probably need to take specific actions for your system, eg. when in ARM state and the system receives a door intrusion event you probably want to start the siren as well (among many other things potentially).

Invalid state transitions will fall into the last overload which works as a wildcard for all the other possible combinations of states and events due to the template argument parameters. Notice how we used `auto` instead of explicitly specifying the template parameters by using the abbreviated function template feature introduced with C++20.

```cpp
struct transitions {
    std::optional<State> operator()(const disarm &, const authenticate &)
    {
        // Do something
        return arm{};
    }

    std::optional<State> operator()(const arm &, const authenticate &)
    {
        // Do something
        return disarm{};
    }

    std::optional<State> operator()(const alarm &, const authenticate &)
    {
        // Do something
        return disarm{};
    }

    std::optional<State> operator()(const arm &, const door &)
    {
        // Do something
        return alarm{};
    }

    // For all the other non valid transitions return std::nullopt
    std::optional<State> operator()(const auto &, const auto &)
    {
        // Invalid state transition
        return std::nullopt;
    }
};
```

Somewhere in your code (could be a class `AlarmFsm` that encapsulates the FSM logic) you will have to implement a similar function that accepts an `Event` and reacts approprietaly depending on whether or not a new transition occured. For example, when there is a new state transition you might need to notify other threads or log the event into a database. How the code reaches this point with an event trigger is out of scope for this post.

*Note:* Don't forget to make your interface thread-safe if `dispatch` is going to be called from multiple threads.

```cpp
void dispatch(const Event &event)
{
    const auto new_state = std::visit(transitions{}, state, event);
    if (new_state.has_value()) {
        state = *std::move(new_state);
        // Do other stuff, eg. logging event in DB or notify other threads
    }
};
```

## Functional approach

There is also a functional approach that we can take with `std::visit`. We could use the `overload pattern` in order to be able to use lambdas instead of overloading the call operator for the `transitions` struct.

To achieve that we are defining a struct called `overloaded` which takes advantage of variadic inheritance, so it derives the call operators for all of the types that has been created with. Basically it brings into overloaded's namespace all of the call operators.

*Note:* If you are compiling with C++17 you will also need the explicit deduction guide for `overloaded` on line 6.

```cpp
// Make all call operators from the base class available to the derived class
template<class... Ts>
struct overloaded : Ts... { using Ts::operator()...; };

// Explicit template deduction guide - needed for C++17
template<class... Ts> overloaded(Ts...) -> overloaded<Ts...>;
```

Putting all of the above together allows us to implement the state machine as follows:

```cpp
const auto new_state = std::visit(
    overloaded{[](const disarm &state,
                  const authenticate &event) -> std::optional<State> {
                   // Do something
                   return arm{};
               },
               [](const arm &state,
                  const authenticate &event) -> std::optional<State> {
                   // Do something
                   return disarm{};
               },
               [](const alarm &state,
                  const authenticate &event) -> std::optional<State> {
                   // Do something
                   return disarm{};
               },
               [](const arm &state,
                  const door &event) -> std::optional<State> {
                   // Do something
                   return alarm{};
               },
               [](const auto &state,
                  const auto &event) -> std::optional<State> {
                   // Invalid state transition
                   return std::nullopt;
               }},
    state_, event)
```

## Conclusion

We presented an alternative way of handling state machines with `std::visit` and `std::variant`. You should consider this method if your transition matrix is complex with many states and events. Of course, the abstraction provided by `std::variant` and `std::visit` comes at a cost, so if performance is your top priority you should measure before deciding the implementation.

You may find the code presented in this blog at [compiler explorer ](https://godbolt.org/z/c475jj43q)

## Extra Resources

If you want to learn more about `std::visit`, here are some resources: 

- [C++ Visitor Design Pattern - Part 4 of 4 - std::visit and std::variant version](https://www.youtube.com/watch?v=NhhvJkBGUlk)
- [C++ Weekly - Ep 440 - Revisiting Visitors for std::visit](https://www.youtube.com/watch?v=et1fjd8X1ho&t=136s)
- [Breaking Dependencies - The Visitor Design Pattern in Cpp - Klaus Iglberger - CppCon 2022](https://www.youtube.com/watch?v=PEcy1vYHb8A)
