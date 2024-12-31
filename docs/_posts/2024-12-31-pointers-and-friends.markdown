---
layout: post
title:  "Pointers and Friends"
date:   2024-12-31 15:15:43 +0100
---
# Pointers
Now after integers and weird array initialisation, let's talk about pointers, references and iterators. First off, pointers point to memory which we can use to reference objects and keep track of them without copying them around. They can be compared, added to and subtracted from and be assigned fix addresses. References reference objects as well, hence the name.
{%highlight cpp%}
int x = 4;
// Pointer to an integer
int* px = &x;
// Reference to an integer
int& px = x;
{%endhighlight%}

This may seem like two things that do the same, making one of them redundant (in fact, references are just pointers with some restrictions), but they have a few notable differences:
- Pointers allow pointer arithmetic while references do not
{%highlight cpp%}
int[] x = {1, 2, 3};

int* px = &x; // Now px points to 1
int* p2x = px + 1; // Now p2x points to 2
{%endhighlight%}

Note that we can let the array decay to a pointer
- Pointers can be null, or point to a specifically assigned address while references can neither
{%highlight cpp%}
// The following are both 'legal'
int* px = nullptr;
int* py = (int*)3000;

// These are definitely NOT
int& rx = nullptr;
int& ry = (int*)3000;
{%endhighlight%}
- Pointers can be reassigned, while references always point to the same object, even if you try reassigning them
{%highlight cpp%}
int x = 10;
int y = 25;

int* px = &x;
px = &y; // Absolutely fine

int& rx = x;
rx = y; // Does not compile 
{%endhighlight%}
- Pointers can be dangling, references must always be valid and act as a sort of alias
{%highlight cpp%}
int* x = new int(5);
delete x;
int y = *x; // Bad idea, but compiles
{%endhighlight%}
-  Pointers have two kinds of ```const```; the following two pointers have very different semantics
{%highlight cpp%}
int x = 4;

const int* y = &x;
int* const z = &x;
{%endhighlight%}
The pointer ```y``` cannot be reassigned to point somewhere else but can modify the value of ```x``` while ```z``` can be reassigned to point somewhere else but cannot modify the memory it points to.

As such, references are pretty much pointers with a lot less footguns because they impose restrictions that guarantee some safety.
### A fun fact about null pointers
We can make a null pointer in three different ways
{%highlight cpp%}
int* px = nullptr; // since C++11
int* yx = NULL; // old but gold
int* zx = 0; // don't
{%endhighlight%}
In addition to this, while the name and the literal ```0``` may suggest it, the memory representation of a nullpointer is not necessarily a bitpattern of only zeroes; it is whatever can be taken as an invalid pointer on the system your compiler targets. This means the following may not print anything.
{%highlight cpp%}
#include <iostream>
int main() {
    int x = 0;
    int* px = 0;

    if ((int)px == x ) {
        std::cout << "Happens to be zero" << std::endl;
    }
    return 0;
}
{%endhighlight%}
In addition to that again, the ```NULL``` is a good old C-style macro (one of those ```#define``` boys) but part of the standard and sometimes special-cased by the compiler, just as the literal ```0``` is. 

# Iterators
Another relative of pointers are iterators, they are the ```vec.begin()``` kind of pointers you pass to some standard algorithms. Like pointers we can increment and dereference them, but instead of just moving up by a ```sizeof``` iterators move to the next element in a container.
{%highlight cpp%}
std::list<int> mylist = {10, 20, 30};
auto it = mylist.begin();
++it;  // Move to the next element
{%endhighlight%}
Since this list does not necessarily store its elements in a contiguous manner, a simple offset by ```sizeof(int)``` may give us an invalid pointer, but the iterator takes care of how to do this correctly.

Iterators are much like references, but they provide a way to traverse a container and come in multiple flavours:
- Input Iterator: Read-only, can move forward.
- Output Iterator: Write-only, can move forward.
- Forward Iterator: Read/write, can move forward.
- Bidirectional Iterator: Can move both forward and backward.
- Random Access Iterator: Can move freely forward and backward, and supports operations like addition/subtraction and comparisons.


## Problems
While iterators are safer and far more convenient than pointers if you want to traverse a container there still needs to be some caution involved.
-  The ```end()``` iterator points 'past' the last element and cannot be dereferenced.
-  You can insert or delete at some iterators, but this invalidates this iterator and possibly others.
-  Containers can reallocate or move their memory for other reasons, which may invalidate your iterator.
-  Dereferencing both ```begin()``` and ```end()``` of an empty container is undefined behaviour.

# A word about smart pointers
There are three main smart pointers in C++:
- ```std::unique_ptr```: Allocates memory upon creation and frees it when its lifetime ends. It can also not be copied and one must use ```std::move``` to move it which invalidates the original pointer.
{%highlight cpp%}
std::unique_ptr<int> ptr1 = std::make_unique<int>(10);
// std::unique_ptr<int> ptr2 = ptr1; // Compiler error, can't copy unique_ptr

std::unique_ptr<int> ptr2 = std::move(ptr1); // Ownership transferred to ptr2
{%endhighlight%}

- ```std::shared_ptr```: Allocates memory upon creation, is reference counted and atomically frees the memory when the count goes to 0. Each copy increases the count and a variable's lifetime ending decreases the count.
{%highlight cpp%}
std::shared_ptr<int> ptr1 = std::make_shared<int>(20);
std::shared_ptr<int> ptr2 = ptr1;  // Both ptr1 and ptr2 share ownership
{%endhighlight%}
Now the reference count has increased to two.

- ```std::weak_ptr```: Used to observe a reference counted object, to use it you need to lock it by calling ```std::weak_ptr::lock()``` which returns a ```shared_ptr```, which if the reference count is already 0 is null.
{%highlight cpp%}
std::shared_ptr<int> ptr1 = std::make_shared<int>(30);
std::weak_ptr<int> weakPtr = ptr1;  // weak_ptr does not affect reference count

// Convert weak_ptr to shared_ptr to access the object
std::shared_ptr<int> lockedPtr = weakPtr.lock();
if (lockedPtr) {
    std::cout << *lockedPtr << std::endl; // Access the object
} else {
    std::cout << "Object no longer exists!" << std::endl;
}

{%endhighlight%}

Some problems that occur are cyclic references which occur if two or more ```shared_ptr```s point to each other which may result in a memory leak and while the ```shared_ptr``` itself is thread safe, the object it points to may not be. To prevent the cycles a ```weak_ptr``` can be used, but this one may be dangling.

The tremendous advantage of smart pointers is how much easier they make memory management by taking care of deallocation. 

This is it for this month, not quite as odd but I hope informative enough to balance it out. Pointers are important and can be dangerous if misused or inattentive so I thought I would cover them a bit more detailed.