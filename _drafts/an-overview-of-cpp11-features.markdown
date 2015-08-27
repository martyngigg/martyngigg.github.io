---
layout: post
title:  "An overview of C++11 features"
date:   2015-09-25 19:25:01
---

This post was primarily written to review new C+11 with the Mantid team but contains no Mantid-specific code. It was adapted from [these slides](https://isocpp.org/blog/2012/12/c11-a-cheat-sheet-alex-sinyakov) by Alex Sinyakov.

C++11 will feel like a new language in many aspects. New language features include:

* `auto`
* `nullptr` constant
* raw string literals
* range-based for loops
* lambdas
* initializer lists for POD structs
* delegating constructors
* r-value references and move constructors

New standard library features include:

* smart pointers - `shared_ptr`, `unique_ptr`, `scoped_ptr`
* tuple
* initializer lists for containers

auto
----

The `auto` keyword takes the place of a hand-written type and forces the compiler to fill in the details:

```c++
std::vector<int> indexes(1);
auto n = indexes.size();
// vs in C++03
std::vector<int>::size_t n2 = indexes.size();
```

Leaning on the compiler in this manner allows us to be confident that the correct type is being used and not rely
on warnings for invalid type conversions that might not always be emitted. Other advantages of using `auto` include:

* removes duplication of type name e.g.

```c++
auto workspace = data_service.retrieve<WorkspaceType>("foo");
```
* allows lambdas (see later) to be captured into a variable without having to write a hideous function pointer type
* requires a variable to be initialised at the point of definition, e.g.

```c++
auto workspace; // ERROR - impossible to deduce type
auto workspace = Workspace2D(); //
```

Take home message: Use `auto` as much as you can!

### auto type deduction

At first glance `auto` seems like a pretty simple beast - it would seem that the type of the `auto` variable is
the same as the type of the initializing expression. However, that is not always the case due to the possibility of
expressions possessing qualifiers such as `const`, `&` or `*`.

Consider these examples:

```c++
// function returing a reference to a const integer
const int & foo();

auto x = foo(); // type of x is int
const auto cx = foo(); // type of cx is const int
const auto & rx = foo(); // type of cx is const int &
```

Notice that even though the function returns a `const int &`, the `auto` specifier does not automatically acquire these properties. The qualifiers must be applied to the variable definition too. This is especially important to keep in mind when the returning value may be expensive to copy - remember to consider whether you want your variable to be a reference or value type.

nullptr
-------

The best that C++03 had to offer for initialising a null pointer was `NULL`. This had many problems, which all revolve around the fact that `NULL` is really just

```c++
#define NULL 0
```

The fact that `NULL` is actually an integer can lead to problems with overload resolution:

```c++
void foo(const char*); //(a)
void foo(int); //(b)


foo(NULL); // you want (a) but you get (b)
```

This has been rectified in C++11 with the introduction of `nullptr`. It has type conversions to any pointer
type so is safe to use in initialization of any pointer value.

```c++
void foo(const char*); //(a)
void foo(int); //(b)


foo(nullptr); // now does what we want!
```
