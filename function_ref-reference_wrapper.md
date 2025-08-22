---
title: "Make function_ref and reference_wrapper interact together"
document: PxxxxR0
date: today
audience: LEWG
author:
    - name: Jeremy Demeule
      email: <jeremy.demeule@gmail.com>
toc: true
tag: functional
---


# Motivation

With [@P0792R14] `function_ref`{.cpp} was introduced to C++ with a very simple way of
binding stateful function.
The state binding is taken by reference only (const or non-const). To store a
`reference_wrapper`{.cpp} object as a state, this mandate to take a reference to a
`reference_wrapper`{.cpp} also.<br>
This paper tries to improve the situation as `reference_wrapper`{.cpp} is already a
representation of a reference to an object, `function_ref`{.cpp} state binding could,
in this case, be handle it directly.

Given:
```cpp
struct X;
std::string work_on_X(const X&);
std::reference_wrapper<X> some_state();
```

::: cmptable

### Before
```cpp
reference_wrapper<X> rx = some_state();
function_ref<std::string()> f {nontype<work_on_X>, rx};
```

### After
```cpp
function_ref<std::string()> f {nontype<work_on_X>, some_state()};
```

:::

The proposed syntax is simpler to use and does not introduce unnecessary variables.

# Wording

The wording is relative to [@N5014].

Add the following signatures to \[func.wrap.ref.class\] synopsis:
```diff
template<class R, class... ArgTypes>
  class function_ref<R(ArgTypes...) @_cv_@ noexcept(@_noex_@)> {
  public:
    // [func.wrap.ref.ctor], constructors and assignment operators
    [...]
+   template<auto f, class T>
+    constexpr function_ref(nontype_t<f>, reference_wrapper<T>) noexcept;
    [...]
  };
```

Add a definition to \[func.wrap.ref.ctor\]

::: add

> ```cpp
> template<auto f, class T>
>  constexpr function_ref(nontype_t<f>, reference_wrapper<T> state) noexcept;
> ```
> Let `F`{.cpp} be `decltype(f)`{.cpp}.<p>
> _Constraints:_ `@_is-invocable-using_@<F, @_cv_@ T&>`{.cpp} is true.<p>
> _Mandates_: If `is_pointer_v<F> || is_member_pointer_v<F>`{.cpp} is `true`{.cpp},
> then `f != nullptr`{.cpp} is `true`{.cpp}.<p>
> _Effects:_ Initializes `@_bound-entity_@`{.cpp} with `addressof(state.get())`{.cpp},
> and `@_thunk-ptr_@`{.cpp} to address of a function `@_thunk_@`{.cpp} such that
> `@_thunk_@(@_bound-entity_@, @_call-args_@...)`{.cpp} is expression equivalent to
> `invoke_r<R>(f, static_cast<cv T&>(@_bound-entity_@), @_call-args_@...)`{.cpp}.

:::

# Acknowledgements

The authors would like to thank:
