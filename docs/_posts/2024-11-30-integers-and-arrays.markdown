---
layout: post
title:  "Integers and Arrays"
date:   2024-11-30 18:30:28 +0100
---
In this blog I will write about and look into odd and/or interesting behaviour I encounter while learning about and writing in C++. I expect there to be more than this single post in the future, so let us begin.
## Integers
Beginning at the basics, probably the most important set of types in the C++ language are the integers.
The language offers a vast array of different integers:
- char
- short
- int
- long
- long long

Each of these may be signed - e.g. `signed int` - or unsigned - e.g. `unsigned short` - and all of them, except `char` whose signedness is implementation defined, are signed by default if unqualified.
A quirk of the sign of `char` being implementation defined means the following
{%highlight cpp%}
#include <iostream>
int main() {
    char c;
    std::cin >> c;
    if (c < 0) {
      char bad_idea = c / 0;
      return 1;
    }
    return 0;
}
{%endhighlight%}
can either compile and crash, compile and work just fine, or refuse to compile depending on which compiler you happen to use at that moment and what number you enter.

For the aforementioned types the standard only defines a minimum size with each larger integer being at least as large as the one before:

| type      | minimum size |
| --------- | ------------ |
| char      | 8 bits       |
| short     | 16 bits      |
| int       | 16 bits      |
| long      | 32 bits      |
| long long | 64 bits      |

Now the factoid of interest in this part concerns overflow. An integer overflow is generally what happens when arithmetic creates a value greater than the maximum representable value using the number of bits of a type.
{%highlight cpp%}
unsigned int a = std::numeric_limits<unsigned int>::max();
unsigned int b = a + 1;
{%endhighlight%}
This is an example of integer overflow and the standard defines what happens in this scenario as unsigned integer overflow is defined. The standard says unsigned integers "wrap around", meaning in the above `b` will be 0. But changing the example to
{%highlight cpp%}
int a = std::numeric_limits<int>::max();
int b = a + 1;
{%endhighlight%}
gives us our first example of undefined behaviour (If we ignore the division by zero in the very first snippet).
Even though the standard does define overflow for unsigned integers, for signed integers overflow is undefined. This enables a number of optimisations but also some very interesting behaviour.
{%highlight cpp%}
#include <iostream>
#include <limits>
int main() {
  unsigned int a;
  std::cin >> a;
  int x = a;
  int b = x + 1;

  if (b < 0) {
      std::cout << "negative" << std::endl;
      return 1;
  }

  return 0;
}
{%endhighlight%}
This program includes undefined behaviour. It could either print out `negative`, immediately close after reading a number as the compiler assumed no undefined behaviour exists and optimized away the check, the integer assignments and basically the whole program, or do the classic "format your hard-drive".

## Arrays
Assigning integers to variables quickly gets very annoying when all you need is a bunch of integers. This issue is addressed by arrays - a contiguous slice of memory storing a specified number of elements side by side. 
We can create arrays either with constant size
{%highlight cpp%}
int array[10];
array[5] = 1;
{%endhighlight%}
or with a variable amount of elements
{%highlight cpp%}
int s;
std::cin >> s;
int array[s];
array[0] = 5;
{%endhighlight%}
and even with multiple dimensions
{%highlight cpp%}
int array[10][20] = {0};
{%endhighlight%}
Notice the `{0}`. You might know its purpose, setting the array to all zeroes. In C++ we can also accomplish this using the `{}` and it even works for arrays of arrays
{%highlight cpp%}
int array[3][4] = {};
{%endhighlight%}

But I must admit, I lied a little just now. `{}` and `{0}` are not quite the same. The following
{%highlight cpp%}
int n, m;
std::cin >> n >> m;
int array[n][m] = {};
n = 10;
m = 50;
int array2[n][m] = {0};
{%endhighlight%}
initialises both arrays to all zeroes just fine but 
{%highlight cpp%}
int n, m;
std::cin >> n >> m;
int array[n][m] = {0};
{%endhighlight%}
results in garbage entries in part of the array. I could not find where in the standard this difference is defined, but my testing found that the latter only reliably sets the first value of the array to zero and all other values are up to whether we have written to this part of the stack or not, as my OS seems to zero-initialise the stack.

I hope this brought you some value, in the following months I will try to explore other oddities and things I bump into and plan on adding new posts to this blog.