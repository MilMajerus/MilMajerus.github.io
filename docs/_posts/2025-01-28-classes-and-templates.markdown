---
layout: post
title:  "Classes and Templates"
date:   2025-01-28 12:23:00 +0100
---
New year; new post. 

C++ was once naught but C with classes. It has certainly evolved since then,
but those classes are still around and allow object orientation to your hearts content. 
Templates, the meta-programming of C++, are probably in combination with classes one of if not the most powerful
feature C++ possesses. While producing often cryptic errors, they allow the programmer to write generic code, reducing duplication, and perform
what some might call compile-time magic.

## Classes
As a quick catchup, this is a class
```cpp
class Something {
  private:
    int private_x;
  public:
    float public_y;
    Something() {
      private_x = 10;
      public_y = 2.5;
    }
}
```
It has a private integer and a public float and constructor.
```cpp
Something obj();

obj.public_y = 10.0; // Absolutely fine
obj.private_x = 2; // Will not compile
```
Since `private_x` is `private`, we cannot assign to it. But there is another problem. What does the following do?
```cpp
Something obj();
```
It looks like we are creating an instant called `obj` but this is a function declaration of a function `obj` that returns `Something`.
The fix looks like this
```cpp
Something obj{};

obj.public_y = 10.0;
```
### Implicit Member Functions
C++ is nice, it wants to help. Sometimes this help is very annoying. 
```cpp
class Wrapper {
  int* ptr; // private by default
  public:
    Wrapper(): ptr(new int(10)) {}
    ~Wrapper() { delete ptr }
}
```
This probably looks fine at first, we create a constructor and a destructor. 

Let us say we want to copy the Wrapper:
```cpp
Wrapper a;
Wrapper b = a;
```
What should happen here? We create a `Wrapper a` and then assign it to `Wrapper b`. In C++ this means we implicitly copy the data. This means we construct `b` using a copy of `a`.
For this there exists a so-called copy constructor, but writing one for every class quickly becomes annoying, so the compiler helps us out and provides one for us.

This is where the problem happens. We shallow copy the pointer to the number `10` and then at the end of the scope the destructor gets called for both `a` and `b`, which means we are deleting the same allocation twice!
The only way to fix this is by either specifying a copy constructor and copy assignment operator as-well, or using some smart pointer whose copy constructor does a deep copy instead of `int* ptr`.

## Templates
To the big guns! 

Templates can be hard to get right, but conceptually, they are no beast.
```cpp
template<typename T>
class Vector {
  public:
    T x;
    T y;
}
```
Look at this! A generic vector which allows us to create a vector of integers, a vector of floating point numbers, a vector of vectors and many more.
```cpp
Vector<int> veci {};
Vector<float> vecf {};
```
When instantiating an object, the compiler takes the `typename` in between angle brackets (`int` in the case of `veci`) and basically pastes is wherever `T` appears in the definition of the class. We can even calculate factorials at compile time: 
```cpp
template<int N>
struct Factorial {
  static const int value = N * Factorial<N - 1>::value;
};
template<>
struct Factorial<0> {
  static const int value = 1;
};
```
C++ also allows us to do what it calls "specialization", or like above, for the case `Factorial<0>` provide a specific alternate definition for that combination of template parameters.
```cpp
template<typename T>
void foo(T) { /* general template */ }

template<>
void foo<int>(int) { /* specialization for int */ }

void foo(int) { /* overload for int */ }
```
But note that for specialization to happen, we must still use `template<>`, otherwise we perform function overloading, which may or may not yield what was intended.

### SFINAE
Probably one of the most insane abbreviations I have heard to date when it comes to features of programming languages: "Substitution Failure Is Not An Error".

This means what it says. When the compiler cannot paste your type into a definition the template is ignored instead of generating an error.

```cpp
template<typename T, typename = std::enable_if_t<std::is_integral_v<T>>>
void foo(T) { /* only works for integral types */ }

void foo(int); // overload

foo(1.0) // SFINAE
```
Notice how `1.0` is not an integral type and thus the substitution fails. But, since it can be implicitly converted to an integer and we call the overload, this compiles and most likely produces what you might expect.


From double deletion to unreadable template errors there is a lot of pain to be had. Classes and templates can be very powerful and supremely useful. Oddities like SFINAE allow advanced compile-time magic that sometimes devolves into the unreadable, but compiles and performs remarkably well. 
It is thus important to know about the above problems as they can be a massive pain to debug. 