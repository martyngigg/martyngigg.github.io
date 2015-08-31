---
layout: post
title:  "An overview of C++11 features"
date:   2015-09-25 19:25:01
---

This post was primarily written to review new C+11 with the Mantid team but contains no Mantid-specific code. It was adapted from [these slides](https://isocpp.org/blog/2012/12/c11-a-cheat-sheet-alex-sinyakov) by Alex Sinyakov.

C++11 will feel like a new language in many aspects. New language features include:

* `auto`
* `nullptr` constant
* range-based for loops
* initializer lists
* lambdas
* delegating constructors
* r-value references and move constructors
* + lots more!

Some new standard library features include:

* smart pointers - `shared_ptr`, `unique_ptr`, `scoped_ptr`
* tuple
* initializer lists for containers

Auto
----

The `auto` keyword takes the place of a hand-written type and forces the compiler to fill in the details:

```c++
std::vector<int> indexes(1);
auto cend = cend(indexes);
// vs in C++03
std::vector<int>::const_iterator cend03 = indexes.end();
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

### Auto type deduction

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

Range-based for loops
---------------------

Iterating around containers in C++03 required a lot of typing:

```c++
vector<int> v(5, 10);
for(vector<int>::iterator itr = v.begin();
    itr != v.end(); ++itr) {
  cout << "Value=" << *itr << std::endl;
}
```

The future in C++11 and beyond is far cleaner:

```c++
vector<int> values(5, 10);
for(auto value : values) {
  cout << "Value=" << value << std::endl;
}
```

Here `auto` is combined with a new syntax to for specifying the range. The type of
the loop variable `value` is the same as the element type of the container. In this
case the loop variable is neither a reference or a pointer so the values in the
container cannot be changed. To access a reference to an element then simply specify
that with your type:

```c++
vector<int> values(5, 10);
for(auto &value : values) {
  value += 1;
  cout << "Value=" << value << std::endl;
}
```

Similarly, if you want a non-modifiable reference to the element then specify the type
as `const auto &`.

Initializer lists
-----------------

Let's say you're writing a test and you need to initialize a `vector` with a list of
different values. In C++03 this was more painful than it should be and in many
circumstances adds to the usage of plain C arrays that have nice syntax for initializing
from a fixed set of values. You could also have opted to use a library such as
[`boost::assign`](doc/libs/1_59_0/libs/assign/doc/index.html).

The future is now brighter with the advent of `std::initializer_list` and *uniform initialization* where we can now do:

```c++
int a[] = {1,2,3,4,5};
std::vector<int> v{1,2,3,4,5};
```

The brace-initialization syntax can also be nested, which is useful for associative
containers:

```c++
map<int, string> states{
  {1, "Off"},
  {2, "On"}
};
```

As will always be the case the types specified in the container must be constructable
from each element in the brace initializer. Furthermore, any class type that has a
constructor that accepts a `std::initializer_list` argument can take advantage of this type of syntax.

Lambdas
-------

As with many things in C++11 lambdas don't techincally offer you anything above
what was possible with C++03. They do however offer much cleaner and more concise syntax
for defining unnamed function objects that can capture variables in the current scope
(basically they are [closures](https://en.wikipedia.org/wiki/Closure_(computer_programming))).


Take, for example, the case where you wish to apply a transformation to a sequence of data using
an STL algorithm but the transformation is not something covered by a basic mathematical function
such as `sqrt`. In C++03 this requires a custom `struct`:

```c++
// ------------------- C++03 --------------------
struct _trans {
  _trans(const double scale) : m_scale(scale) {}
  double operator()(double x) const { return m_scale *sqrt(x) / M_PI; };
  double m_scale;
};

using boost::assign::list_of;
std::vector<double> v = list_of(1.0)(2.0)(3.0)(4.0)(5.0);
const double scale = scaleFactor();
std::transform(v.begin(), v.end(), v.begin(), _trans(scale));
```

Lambdas allow us to drastically reduce the amount of code required to do the same thing:

```c++
// ------------------- C++11 --------------------
std::vector<double> v{1.0, 2.0, 3.0, 4.0, 5.0};
const double scale = scaleFactor();
std::transform(begin(v), end(v), begin(v),
                 [scale](const double &x) { return scale *sqrt(x) / M_PI; });
```

Internally the compiler actually generates an anonymous `struct` that does exactly what we would
have written ourselves but leaning on the compiler allows us to reduce the amount of
code we have to maintain and avoid bugs.

### Lambda capture

In the example above the definition of the lambda looks mostly like a regular function definition
with the name replaced by the square bracket notation - this bracket notation is called the *capture*.
It allows the generated struct to reference the environment that it is defined in.

The following scenarios define what can be captured:

* `[a,&b]` where `a` is captured by value and `b` is captured by reference.
* `[this]` captures the `this` pointer by value
* `[&]` captures all *automatic* variables in the body of the lambda by reference
* `[=]` captures all *automatic* variables in the body of the lambda by value
* `[]` captures nothing

One of Scott Mayers' tips is to avoid the two "capture all" scenarios and implicitly specify the required
variables to limit the scope that the lambda has to affect its environment.

Delegating constructors
-----------------------

Writing a class with multiple constructors often requires them to share common setup code such as defining
member variables. This is generally done by writing a separate private member function to handle the
setup without repeating code. While the private method eliminates the duplicate code it has other problems:

* other member functions might accidentally call `init()`, which causes unexpected results.
* after we enter a class member function,  all the class members have already been constructed. It's too
  late to call member functions to do the construction work of class members.

C++11 introduces the concept of *delegating constructors* where a single constructor takes ultimate responsibility
for initializing the object and other constructors can call this construtor:

```c++
class A{
public:
  A(): A(0){}
  A(double i): A(i, 0){}
  A(double i, double j) {
    num1 = i;
    num2 = j;
    average = 0.5*(num1+num2);
   }
private:
  double num1;
  double num2;
  double average;
};
```
