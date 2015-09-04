---
layout: post
title:  "An overview of C++11 features"
date:   2015-09-04 22:45:01
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

Move semantics & r-value references
-----------------------------------

Consider a class that is expensive to copy and a factory function for an object of such a class:

```c++
SquareMatrix create(size_t nrows, bool addone) {
  if (addone) {
    SquareMatrix s1(nrows + 1);
    return s1;
  } else {
    SquareMatrix s2(nrows);
    return s2;
  }
}

class SquareMatrix {
  SquareMatrix(size_t nrows) : m_data(allocate(nrows)), m_nrows(nrows) {}
  ~SquareMatrix() { delete[] m_data; }
  // Copy constructor
  SquareMatrix(const SquareMatrix &other)
    : m_data(allocate(other.m_nrows)), m_nrows(other.m_nrows) {
    using std::copy;
    // Use assignment operator
    *this = other;
  }
  // Copy assignment operator
  SquareMatrix &operator=(const SquareMatrix &rhs) {
    using std::copy;
    if (this != &rhs) {
      // size change means new memory allocation
      if (m_nrows != rhs.m_nrows) {
        delete[] m_data;
        m_data = allocate(rhs.m_nrows);
        m_nrows = rhs.m_nrows;
      }
      copy(rhs.m_data, rhs.m_data + m_nrows * m_nrows, m_data);
	}
    return *this;
  }

private:
  double *allocate(const size_t nrows);

  double *m_data;
  size_t m_nrows;
};
```

Let's examine two uses:

```c++
SquareMatrix m1(4);
SquareMatrix m2 = m1;
SquareMatrix m3 = create(3, false);
```

In both assignments a copy of the object will be performed. The first assignment requires a copy because the original and new objects must
be able to survive independently afterwards. However, the second example uses the temporary object returned from the `create` function
and copies this in to the new matrix object. In this case the temporary is then destroyed after the assignment as it is no longer needed. It
would therefore be better if we could somehow detect that the assignment/construction comes from a temporary and do something different
and more efficient in these cases, e.g. move the data rather than copy it.

### Rvalue references

C++03 has always had the concept of an *lvalue* and an *rvalue*. Roughly speaking:

* an *lvalue* is an expression that refers to a memory location and allows us to take the address of that memory location via the `&` operator
* an *rvalue* is an expression that is not an *lvalue*.

In our example above `m1` is an *lvalue* and the object returned from the `create` function is an *rvalue* because we have no handle that
can be used to find out what its address is. C++11 introduces the concept of "rvalue references" to allow us as the programmer to write
function overloads that are able to distinguish between whether they have been called by with an lvalue or rvalue - exactly what we
need to implement move semantics!

To keep consistency with a C++03 reference denoted by a single ampersand (`&`), technically an lvalue reference, rvalue references are denoted
by a double ampersand (`&&`). Let's augment our `SquareMatrix` example with overloads that understand rvalue references (for the sake of brevity
I only show the new code below):

```c++
// Move constructor
SquareMatrix::SquareMatrix(SquareMatrix &&other)
    : m_data(nullptr), m_nrows(other.m_nrows) {
  using std::swap;
  cerr << "Move constructor\n";
  swap(m_data, other.m_data);
  m_nrows = other.m_nrows;
}

// Move assignment operator
SquareMatrix &SquareMatrix::operator=(SquareMatrix &&rhs) {
  using std::swap;
  cerr << "Move-assignment operator\n";
  swap(m_data, rhs.m_data);
  m_nrows = rhs.m_nrows;
  return *this;
}
```

The key difference to notice about the overloads is that the references are not `const` so that the "other" object can be changed
however we wish, afterall it's considered a temporary value so no one else can hold a reference to it (not quite true). This freedom enables
us to be a little more brutal and rather than copying values from the other object we simply exchange the internal array pointers. It is important
to remember that the temporary object will still be deallocated and must be left in a valid state so that this can occur.

### std::move

Now that we have our new, more-efficient, overloads it would be nice to clean up the duplicated code between the two methods and implement
the move constructor in terms of the move-assignment operator. Naively we might do this:

```c++
// Move constructor
SquareMatrix::SquareMatrix(SquareMatrix &&other)
    : m_data(nullptr), m_nrows(other.m_nrows) {
  using std::swap;
  cerr << "Move constructor\n";
  *this = other;
}

// Move assignment operator
SquareMatrix &SquareMatrix::operator=(SquareMatrix &&rhs) {
  using std::swap;
  cerr << "Move-assignment operator\n";
  swap(m_data, rhs.m_data);
  m_nrows = rhs.m_nrows;
  return *this;
}
```

This actual causes the move-constructor to call the **copy**-assignment operator and not, as required, the move-assignment operator. The reason is that
even though the type of the `other` parameter is an *rvalue* reference, `other` itself is actually an *lvalue* - we can take its address, therefore by definition
it is an *lvalue*. Trying to call an assignment operator with an lvalue will force call to the copy variant and not the move variant. To indicate to the compiler
that we want to transform from an lvalue to an rvalue reference we use the new `std::move` function:

```c++
// Move constructor
SquareMatrix::SquareMatrix(SquareMatrix &&other)
    : m_data(nullptr), m_nrows(other.m_nrows) {
  using std::swap;
  cerr << "Move constructor\n";
  *this = std::move(other);
}
```

It is important to understand that `std::move` simply casts an lvalue reference to an rvalue reference - no movement actually happens, despite the name.

The full code for the above example can be downloaded from [here](https://github.com/martyngigg/sandbox/tree/master/cpp/move-sematics).

Summary
-------

This has covered a fraction of the new C++11 standard and not touched at all on C++14 or indeed C++17! There are many great resources for reading more
about these topics. Some I have used are:

* [Effective Modern C++: 42 Specific Ways to Improve Your Use of C++11 and C++14 - Scott Meyers](http://shop.oreilly.com/product/0636920033707.do)
* [Slides from isocpp blog - Alex Sinyako](https://isocpp.org/blog/2012/12/c11-a-cheat-sheet-alex-sinyakov)
* [Herb Sutter's blog](http://herbsutter.com/elements-of-modern-c-style/)
