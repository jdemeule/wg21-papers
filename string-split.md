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
There was a question about the reference type of the operation and the question was resolved by [@P1989R2] that introduced range constructor on `std::string_view`{.cpp}.
Indeed, the range constructor of `std::string_view`{.cpp} allows very simple usage like these ones:

```cpp
std::size_t fct_on_str(std::string_view some_str);


std::string str = "the quick brown fox";
auto w1 = str | ranges::views::split(' ') | ranges::to<std::vector<std::string_view>>();
auto w2 = str | ranges::views::split(' ') | ranges::to<std::vector<std::string>>();

auto s1 = str | ranges::views::split(' ') | ranges::views::transform(fct_on_str);

constexpr auto str_view = "the quick brown fox"sv;
auto s2 = str_view | ranges::views::split(' ') | ranges::views::transform(fct_on_str);
```

A `std::string`{.cpp} or a `std::string_view`{.cpp} split returns a range of something that could be handled as a string (here an object that is implicitly convertible to a `std::string_view`{.cpp}).
This is very coherent of what a developer expects when doing a split.
This simplifies the code, reduces the learning overhead, it is conceptually right.
In the same way, materializing the string should be as natural as constructing it.

However, with [@P2499R0], `std::string_view`{.cpp} range construction is now explicit, which made the previous example erroneous. Some additional steps are now needed to be equivalent:

```cpp
auto s1 = str
   | ranges::views::split(' ')
   | ranges::views::transform([](auto s) { return std::string_view(s); })
   | ranges::views::transform(fct_on_str);

constexpr auto str_view = "the quick brown fox"sv;
auto s2 = str_view
   | ranges::views::split(' ')
   | ranges::views::transform([](auto s) { return std::string_view(s); })
   | ranges::views::transform(fct_on_str);
```

Same, when we need to store the result to a `vector`{.cpp}:
```cpp
auto w1 = str
   | ranges::views::split(' ')
   | ranges::views::transform([](auto s) { return std::string_view(s); })
   | ranges::to<std::vector<std::string_view>>();
auto w2 = str 
   | ranges::views::split(' ')
   | ranges::views::transform([](auto s) { return std::string(std::from_range_t{}, s); })
   | ranges::to<std::vector<std::string>>();
```

After some research, it seems C++ will be one of the few languages (or the only one) that splitting a string does not return a string.
Some examples of string splitting on some other languages:

In Rust:
```rust
let v: Vec<&str> = "a,b,c".split(',').collect();
assert_eq!(v, ["a", "b", "c"]);
```

In Python:
```python
'a,b,c'.split(',') == ['a', 'b', 'c']
```

In Go:
```go
a := strings.Split("a,b,c", ",")
b := []string{"a", "b", "c"}
reflect.DeepEqual(a, b)
```

In C#:
```C#
string s = "a,b,c";
string[] subs = s.Split(',');
Enumerable.SequenceEqual(subs, new string[]{"a", "b", "c"})
```

In Java:
```Java
String s = "a,b,c";
String[] expected1 = new String[] { "a", "b", "c" };
assertArrayEquals(expected1, s.split(","));
```

This paper attempts to get back to the original behavior without breaking the fix done by [@P2499R0].

# Design

First we consider the split algorithm on a string object is not an edge case. So we could keep the information that the split is done on a string-like object (`std::string`{.cpp}, `std::string_view`{.cpp}) and provide an implicit conversion on the result of the string.

This could be done by modifying the `split::iterator::value_type`{.cpp} to return an object that is implicitly convertible to `string_view`{.cpp} or a `subrange`{.cpp} as before.

# Proposal
This paper proposes to introduce some helper traits to detect what is a `@_string-like_@`{.cpp} type, a concept to activate the feature and a dedicated `value_type`{.cpp} for `split::iterator`{.cpp} in that case.

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
`@_string-like_@`{.cpp} is a concept to detect a string object 
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
struct string_detector_traits<std::basic_string_view<Char, Traits>> {
   using type = Traits;
};
template <class Char, class Traits>
struct string_detector_traits<std::basic_string<Char, Traits>> {
   using type = Traits;
};
template <class R>
struct string_detector_traits<ref_view<R>> : public string_detector_traits<R> {};
template <class R>
struct string_detector_traits<owning_view<R>> : public string_detector_traits<R> {};
```

_Editor note: this trait could be a customization point for string-like objects outside those the standard provides._

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

Another approach is to provide dedicated string split instead of the general `ranges::views::split`{.cpp}.
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

# Acknowledgments

Thanks to Raoul Borges and Loïc Joly for the feedback.<br>
Thanks to Benoit De Backer for the corrections.
