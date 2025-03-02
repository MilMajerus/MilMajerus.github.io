---
layout: post
title:  "Weird Containers"
date:   2024-03-02 21:43:12 +0100
---
In many ways C++, the language, is weird. Naturally, this carries over into C++, the library. Over its long history, many mistakes have been made, and many fixes have been created, but backwards compatibility lets us use them all.


## ```priority_queue```'s Backwards Comparisons
Starting with the mild, ```std::priority_queue``` is a max-heap. This may be contrary to your expectations, but is purely a design choice. Sadly, this is annoying for implementing algorithms that require a min-heap if you're unaware. The fix is rather simple, but somewhat unintuitive.
```cpp
std::priority_queue<int, std::vector<int>, std::greater<int>> min_pq;
```
Using ```std::greater``` as the third template argument gives us a min-heap. That something names ```greater``` gives us a min-heap is a bit weird, but such is the standard.
## Maps
The ```std::map<K, V>``` in C++, as you may know, is a binary tree from a set of key ```K``` to values ```V```.
If we wish to insert an element into the map, we simply need to ```emplace(K, V)``` it. Yet, if we don't know whether a key has already been assigned to, emplace, does not do what we expect it to.
```cpp
std::map<int, std::unique_ptr<Resource>> resources;
resources.emplace(42, std::make_unique<Resource>());  // Works
resources.emplace(42, std::make_unique<Resource>());  // Silently does nothing!
```
Emplace doesn't replace existing elements, even though it looks like it should. The second call creates and immediately destroys the Resource.

The fix to that is to use ```inset_or_assign(K, V)``` (since C++17) or first check whether the element already exists.
```cpp
resources.insert_or_assign(42, std::make_unique<Resource>());

// Or check existence first:
auto [it, inserted] = resources.try_emplace(42, nullptr);
if (!inserted) {
    it->second = std::make_unique<Resource>();
}
```

## ```forward_list```'s Missing Size
Like ```std::list```, ```std::forward_list``` is a linked list, but with the difference of only supporting forward iteration. This saves on space while still allowing fast insertion and removal from anywhere in the list.
Unlike ```std::list``` (which since C++11 has required O(1) size retrieval), ```std::forward_list``` has no ```.size()``` method. The only fix to this is either calculate it manually, or create a wrapper.
```cpp
// Either track size manually:
auto count = std::distance(flist.begin(), flist.end());

// Or use a wrapper:
template <typename T>
class TrackedForwardList {
    std::forward_list<T> list;
    size_t size_ = 0;
public:
    void push_front(const T& val) {
        list.push_front(val);
        size_++;
    }
    size_t size() const { return size_; }
};
```

## ```deque```'s Iterator Annihilation
While Deques maintain iterator validity for end operations, they nuke all iterators when inserting in the middle.
```cpp
std::deque<int> d {1, 2, 3, 4};
auto it = d.begin() + 2;   // Points to 3
d.push_front(0);           // it still valid
d.insert(it, 99);          // Invalidates ALL iterators!
```
The best solution I have found is to recompute the iterator
```cpp
auto new_it = d.begin() + (it - d.begin());
```

## ```initializer_list`` and temporaries
Temporaries, for the uninitiated, are values that exist only temporarily. Makes sense. As opposed to static or owned values, which are either placed into the data section or the heap of the binary, temporary values can exist on the stack and pointers to them should not be returned from functions. Initializer lists are one place where temporaries pop up.
```cpp
std::vector<const char*> strings {
    "temporary", "danger", "zone"  // Elements point to static data
};

auto bad = []() -> std::vector<std::string_view> {
    return {"these", "pointers", "dangle"};  // UNDEFINED BEHAVIOR
}();
```
Notice how strings is valid as it points to static data in the binaries - usually - ```rodata``` section, while the anonymous function that assigns to  ```bad``` returns a vector of ```std::string_view```s that point to strings on the anonymous function's call stack. This creates dangling pointers/references and thus undefined behaviour.

To remedy this situation, you must ensure the ```std::string_view```s point to static or not-freed allocated memory.
```cpp
// Always copy string literals immediately
std::vector<std::string> safe_strings {
    "permanent", "storage", "achieved"
};

// For views: construct from proper strings
std::vector<std::string_view> safe_views;
for (const auto& s : safe_strings) {
    safe_views.push_back(s);
}
```

## ```[]``` creates life

The ```[]``` operator for ```std::unordered_map``` has the ability to create entries in the map where there are none. This means the following always enters the if statement.
```cpp
std::unordered_map<std::string, int> word_map;
if (word_map["missing"] == 0) {  // Creates entry!
    // Always enters here, even for truly missing keys
}
```
To actually check for existence, there are two ways, the pre C++20 way (and hence the one you will probably have to use) and the good way.
```cpp
// Use find() for existence checks
auto it = word_map.find("missing");
if (it != word_map.end() && it->second == 0) {
    // Actual existing zero
}

// C++20 contains() for simple checks
if (word_map.contains("missing")) { /*...*/ }
```

## The Infamous
Widely regarded as one of the most egregious mistakes of the C++ standard, ```std::vector<bool>``` while keeping true to the promise of keeping all its elements in contiguous memory, is specialized to a bitset.
By bitset I mean that each boolean takes up only one bit and hence ```std::vector<bool>``` only needs one byte per 8 values. While this is great for space savings, it messes with keeping references to individual booleans. 
A ```std::vector<T>``` allows ```T&``` to its individual elements for all ```T``` except ```bool```, for which a reference returns a proxy object, which breaks the expectations set upon every other container.

Luckily, there are three alternatives (two if you discount the inclusion of additional libraries).
```cpp
// Use alternatives:
std::vector<char> bools;         // 1 byte per bool
std::bitset<100> fixed_bits;     // Compile-time size
boost::dynamic_bitset<> dyn_bits;// Runtime-sized bit array
```


## Conclusion

Who could have known that ```contains```, something I regard as essential for containers and especially hashmaps, has only been added to ```std::unordered_map``` in C++20, or that ```std::deque``` destroys some iterators and others not. This is but a taste of the edges you can cut yourself on when working with the STL containers and there is only one solution I can recommend: to keep a tab of the reference open in your browser at all times.