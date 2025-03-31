---
layout: post
title:  "Classes and Templates"
date:   2025-03-31 21:40:00 +0100
---

The Algorithm Module

C++'s standard library algorithms are a marvel of generic programming, letting you manipulate data with a few keystrokes. But lurking beneath their elegant interfaces are oddities that could baffle even the most knowledgeable of us. From iterator invalidation to predicates gone rogue, let’s unravel the quirks that make the algorithm module equal parts powerful and perplexing.

# The Misleading `std::remove`
Consider this common scenario: you want to erase all elements matching a value from a std::vector. You reach for std::remove, write the following, and nothing happens.
```cpp
std::vector<int> numbers = {1, 2, 3, 2, 5};
std::remove(numbers.begin(), numbers.end(), 2);
// numbers is now {1, 3, 5, 2, 5} (maybe?)
```
Despite what its name may suggest  `std::remove` does not erase elements. Instead, it rearranges the range, moving all non-removed elements to the front and returning an iterator to the new "logical" end. To physically erase the elements, you must partner it with erase:
```cpp
auto new_end = std::remove(numbers.begin(), numbers.end(), 2);
numbers.erase(new_end, numbers.end()); // Now actually removes the elements
```

# Predicates
Many algorithms, like std::remove_if or std::sort, accept predicates to customize behaviour. But what if your predicate has state?
```cpp
struct Counter {
    int count = 0;
    bool operator()(int) { 
        return ++count % 2 == 0; // Remove every other element?
    }
};

std::vector<int> nums = {1, 2, 3, 4, 5};
auto it = std::remove_if(nums.begin(), nums.end(), Counter{});
nums.erase(it, nums.end());
```
The predicate Counter is copied during the algorithm's execution. Each copy resets count to zero, leading to undefined behaviour. The standard allows algorithms to copy predicates freely, so stateful predicates are a recipe for chaos. Moral of the story: keep your predicates stateless and pure.

# `std::nth_element`
Imagine you want the 3rd smallest element in a vector. You use std::nth_element, expecting the 3rd position to be correct, but what about the rest?
```cpp
std::vector<int> vec = {5, 7, 3, 1, 9, 4};
std::nth_element(vec.begin(), vec.begin() + 2, vec.end());
// vec might be {1, 3, 4, 7, 9, 5} 
// The 3rd element (index 2) is 4, but the rest are only partially ordered.
```
std::nth_element partially sorts the range such that the nth element is in its correct sorted position, but the elements before and after are merely partitioned—not fully sorted. This makes it efficient for selecting a single element but surprising for those expecting a complete order.

# The necessity of sorting
Some algorithms demand sorted ranges to function correctly. `std::binary_search`, `std::set_union`, and friends assume their inputs are sorted. Feed them an unsorted range, and you’re in for undefined behaviour:
```cpp
std::vector<int> unsorted = {5, 1, 3, 2};
bool found = std::binary_search(unsorted.begin(), unsorted.end(), 3);
// Undefined behavior! The range isn't sorted.
```
The compiler won’t warn you. The code might even "work" sometimes. Always ensure your ranges are sorted before invoking these algorithms.

# The Iterator Invalidation Illusion
Algorithms like `std::transform` or `std::copy` assume their iterators point to valid ranges. But what if you mutate the container mid-operation?
```cpp
std::vector<int> vec = {1, 2, 3, 4};  
auto it = std::remove_if(vec.begin(), vec.end(), [](int x) {  
    vec.push_back(x * 2); // Iterator invalidated
    return x % 2 == 0;  
}); 
```
Remember my post about pointers? The moment you `push_back`, `resize`, or `erase`, iterators can become dangling pointers. The algorithm, oblivious to your recklessness, charges ahead into the void.

# Unique duplicates
Want to remove duplicates? `std::unique` seems like the tool—until you realize it only removes adjacent duplicates:
```cpp
std::vector<int> data = {1, 2, 2, 3, 2, 4};  
auto new_end = std::unique(data.begin(), data.end());  
data.erase(new_end, data.end());  
// data becomes {1, 2, 3, 2, 4}  
```
Notice the second `2`? Even though we called a function with the name `std::unique`, we are left with duplicates. If you wish to use said function and obtain a truly unique set, you must first sort.
```cpp
std::sort(data.begin(), data.end());  
auto new_end = std::unique(data.begin(), data.end());  
data.erase(new_end, data.end());
```

# Lambda Captures and Dangling References
Lambdas can be used in many of the standard algorithms to describe a variety of things, like what to do on each step. But capturing locals by reference is dangerous, look at the following:
```cpp
std::vector<int> numbers = {1, 2, 3, 4};  
int threshold = 3;  

auto it = std::remove_if(numbers.begin(), numbers.end(),  
    [&threshold](int x) { return x > threshold; }  
);  
numbers.erase(it, numbers.end());  
// Safe... unless `threshold` goes out of scope!  

auto dangerous_lambda = [&threshold](int x) { return x > threshold; } ;  
```
Now notice that we may return the latter lambda from a function, where `threshold` will have gone out of scope, giving us a dangling reference.

# Custom Comparators: The Strict Weak Ordering Trap
For some reason custom comparators must enforce so-called strict weak ordering.
```cpp
// A broken comparator that "sorts" even and odd numbers.  
struct BadComparator {  
    bool operator()(int a, int b) const {  
        return (a % 2) < (b % 2); // Even numbers first!  
    }  
};  

std::vector<int> nums = {3, 2, 5, 4};  
std::sort(nums.begin(), nums.end(), BadComparator{});  
// Undefined behavior! Fails strict weak ordering.  
```
he comparator must satisfy:

1. comp(a, a) == false (irreflexivity).
2. If comp(a, b) == true, then comp(b, a) == false (asymmetry).
3. Transitivity.

Violate these, and your program might crash, loop infinitely, or summon nasal demons.

# Undefined transform
`std::transform` lets you map a range, but if your forget initialising the destination, or do not make sure the source and destination are of equal length, you get undefined behaviour.
```cpp
std::vector<int> src = {1, 2, 3};  
std::vector<int> dest;  
dest.reserve(src.size());  

std::transform(src.begin(), src.end(), dest.begin(),  
    [](int x) { return x * 2; }  
);  
// Undefined behavior: Writing to uninitialized `dest`!  
```
Always pair `std::transform` with a back inserter for empty destinations.
```cpp
std::transform(src.begin(), src.end(), std::back_inserter(dest),  
    [](int x) { return x * 2; }  
); 
```
The back inserter does as its name implies, it inserts at the back.

# Execution Policies
C++17 introduced parallel execution policies for algorithms. But tread carefully:
```cpp
std::vector<int> data = {5, 3, 1, 4, 2};  
std::sort(std::execution::par, data.begin(), data.end());  
```
Parallel algorithms are not magic. They require:
1. Thread-safe operations (no shared state in predicates).
2. No iterator/reference invalidation.
3. A compiler/library that actually supports them.

If not careful, they can introduce race conditions or deadlocks.

# Conclusion
`std::algorithms` is great! There is much fun to be had and time to be saved, both in programmer hours and execution time, because let's be honest: most of us will not write a better binary search than the standard one, optimised by hundreds of top-notch engineers. 
With that tremendous power come a few pain points, such as invariants that may not be obvious and the all-too-ubiquitous unintuitive naming conventions, but mastery, or even cursory knowledge, of `std::algorithms` can truly simplify your day-to-day work and convey your intention much clearer than cooking up your own solution. 