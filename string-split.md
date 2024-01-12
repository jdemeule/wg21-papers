---
title: "Result of string | views::split should be usable with string_view"
document: PxxxxR0
date: today
audience: LEWG
author:
    - name: Jeremy Demeule
      email: <jeremy.demeule@gmail.com>
toc: true
tag: ranges
---

# Introduction

[@P2210R2] simplified a lot string splitting.
Thanks to Barry Revzin's work, the contiguous property of a string is kept during split.
There was a question about the reference type of the operation and the question was resolved by [@P1989R2] that introduce range constructor on `std::string_view`{.cpp}.
Indeed, the range constructor of `std::string_view`{.cpp} allow very simple usage like those ones:

```cpp
std::size_t fct_on_str(std::string_view some_str);


std::string str = "the quick brown fox";
str | ranges::views::split(' ') | ranges::to<std::vector<std::string_view>>();
str | ranges::views::split(' ') | ranges::to<std::vector<std::string>>();

str | ranges::views::split(' ') | ranges::views::transform(fct_on_str);

constexpr auto str_view = "the quick brown fox"sv;
str_view | ranges::views::split(' ') | ranges::views::transform(fct_on_str);
```


This is very coherent of what a developer expect when doing split.
For us, if `std::ranges::views::split`{.cpp} is applied on a string like container, the result of it should be compatible with a string like.
This simplify the code, reduce the learn overhead, it is conceptually right.
In the same way, materializing the string should be natural as constructing it.

However, with [@P2499R0], `std::string_view`{.cpp} range construction is now explicit, which made the previous example erroneous. Some additional steps are now needed to be equivalent:

```cpp
str | ranges::views::split(' ') | ranges::views::transform([](auto s) {
      return std::string_view(ranges::data(s), ranges::size(s));
   }) |
      ranges::views::transform(fct_on_str);

constexpr auto str_view = "the quick brown fox"sv;
str_view | ranges::views::split(' ') | ranges::views::transform([](auto s) {
      return std::string_view(ranges::data(s), ranges::size(s));
   }) |
      ranges::views::transform(fct_on_str);
```

Same, when we need to materialize a string:
```cpp
str | ranges::views::split(' ') |
      ranges::views::transform([](auto s) {
         return std::string(std::from_range_t{}, s);
      }) |
      ranges::to<std::vector<std::string>>();
```

After some research, C++ will be one of the few language (or the only one) that splitting a string does not return a string.
This paper try to get back the original behavior without breaking the fix done by [@P2499R0].

# Design

First we consider the split algorithm on string object is not an edge case. So we could keep the information that the split is done on a string-like object (`std::string`{.cpp}, `std::string_view`{.cpp}) and provide an implicit conversion on the result of the string.

This could by done by modifying the `split::iterator::value_type`{.cpp} to return a object that is implicitely convertible to string_view or a subrange as before.

# Proposal
This paper propose to introduce some helper traits to detect what is a `@_string-like_@`{.cpp} type, a concept to activate the feature and a dedicated `value_type`{.cpp} for `split::iterator`{.cpp} in that case.

## split_view::iterator

```cpp
namespace std::ranges {
  template<forward_range V, forward_range Pattern>
    requires view<V> && view<Pattern> &&
             indirectly_comparable<iterator_t<V>, iterator_t<Pattern>, ranges::equal_to>
  class split_view<V, Pattern>::iterator {
  private:
//[…]
  using value_type = std::conditional_t<@_string-like_@<V>, string_like_subrange<iterator_v<V>, V>, subrange<iterator_v<V>>>
//[…]
  };
}
```

## string-like
`@_string_like_@`{.cpp} is a concept to detect a string object 
```cpp
template <class R>
concept @_string-like_@ = std::ranges::contiguous_range<R> && requires { typename string_detector_traits<R>::type; };
```

## string_detector_traits
The `string_detector_traits` is a simple traits to detect string and extract the char traits applied to it:
```cpp
template <class T>
struct string_detector_traits;
template <class Char, class Traits>
struct string_detector_traits<fsl::basic_string_view<Char, Traits>> {
   using type = Traits;
};
template <class Char, class Traits>
struct string_detector_traits<std::basic_string<Char, Traits>> {
   using type = Traits;
};
template <class R>
struct string_detector_traits<ref_view<R>> : public string_detector_traits<R> {};
```

_Editor note: this traits could be a customization point for string-like object outside those the standard provide._

## string_like_subrange
And finally the returned value of split in the case of a `@_string-like_@`{.cpp} object could be:
```cpp
template <class I, class V>
struct string_like_subrange : public subrange<I> {
   using traits = typename string_detector_traits<V>::type;
   using string_view_type = std::basic_string_view<iter_value_t<I>, traits>;
   operator string_view_type() const {
      return {ranges::data(*this), ranges::size(*this)};
   }
};
```


# Alternative

Another approach is to provide dedicated string split instead ofthe general `ranges::views::split`{.cpp}.
A simple implementation of that could be:
```cpp
namespace std::ranges::views {
inline constexpr __adaptor::_RangeAdaptor string_split =
    []<class Char, class Traits, class F>(basic_string_view<Char, Traits> str, F&& f) {
      return split_view{str, std::forward<F>(f)} |
        views::transform([](auto s){
          return basic_string_view<Char, Traits> {ranges::data(s), ranges::size(s)};
        });
    };
}
```
