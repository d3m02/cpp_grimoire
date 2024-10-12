[Back to Basics: Classic STL - Bob Steagall - CppCon 2021 (youtube.com)](https://www.youtube.com/watch?v=tXUXl_RzkAk)

What is "Classic STL"? The C++20 Standard Library can be separated on next sections

|                  |                     |                   |
| ---------------- | ------------------- | ----------------- |
| Language support | General Utilities   | Atomic Operations |
| Concepts         | Containers          | Thread Support    |
| Diagnostics      | Iterators           | Numerics          |
| Strings          | Algorithms          | Time              |
| Ranges           | Input/Output        | Localization      |
|                  | Regular Expressions |                   |
Classic STL - it's mostly about Original(General) Utilities + Containers + Iterators + Algorithms. 

## STL motivation 
Speaking about motivation, we need to take for review what is programming in general. We have some business problem to solve. We begin with input data, read that data and perform computations, generate and write some desired output. 
Data is almost always _collections of elements_. A virtually infinite number of data element types. Each collection of elements has some _representation_ (way of structuring elements), a large number of possible representations. There are many kind of processing (_algorithms_), a very large number of algorithms. 
And so, in any g problem space, the choices are fewer, traditionally, a combinatorial explosion of code - $N_T * N_R * N_A$. 
Goal of STL - achieve smaller number of things we need to worry about -  $N_T + N_R + N_A$. 


## Brief history 
* 1979, Alexander Stepanov begins exploring generic programming (GP)
* 1988, Stepanov and David Musser publish Generic Programming 
	> 'Generic programming centers around the idea of abstracting from concrete efficient algorithms to obtain generic algorithms that can be combined with different data representations to produce a wide variety of useful software'
* 1992, Meng Lee joins Stepanov at HP Research Labs, where his team is experimenting with C and C++
* 1993, Stepanov presents that main ideas at the November WG21 meeting
* 1994, Stepanov and Lee create proposal for WG21 that was accepted later that year 
* 1994-1998, much additional work; adding the original associative containers 
* 1998, first ISO  C++ Standard published 
* 2011, C++11 is published, and with some new containers. 

Stepanov at his first presentation described his principles. He want _comprehensive_ (take all best from APL, Lisp, Dylan, C Library, USL Standard Components; Provide structure and fill the gaps), _extensible_ (orthogonality of the component space, if you change one algorithm - it should require changing container; semantically based interoperability guarantees), _efficient_ (no penalty for generality, complexity guarantees at interface level), _natural_ (C/C++ machine model and programming paradigm, support for built-in data types)


## Overview
### Key principles of classic STL 
Containers - data storage that store collections of elements. 
Algorithms perform operations upon collections of elements. Algorithm not necessary perform operation upon containers, they can work with elements outside containers (like data streams). 
Containers and algorithms are entirely independent. 
And Iterators provide a common unit of information exchange between containers and algorithms. 

STL makes complexity guarantees by specifying interfaces and requirements. 
Containers provide support for: 
* Adding/removing elements 
* Accessing (reading/updating) elements via associated iterators
* A container's iterators understand (and abstract) that container's internal structure.
Iterators:
* Provide access to container elements through well-defined interfaces with strict guarantees
Algorithms:
* Employ the well-defined interfaces provided by iterators 
* Have complexity based on the algorithm itself and the guarantees made by the iterators 


### Containers 
Containers hold a collection of elements. STL containers are implemented using a variety of basic data structures. Each STL container represents a sequence of elements. 
Containers have an internal structure and ordering. We can observe this ordering and sometimes we can control the ordering, in fact in majority containers we don't control ordering and order managed by how container internally operates. 
**Containers own the elements they hold**. Ownership means element lifetime management. Containers construct and destroy their member elements.

Sequence containers - containers order which we can observe and modify
* `vector` 
* `deque` 
* `list`
* `array` (C++11)
* `forward_list` (C++11)
Associative containers - they have order and they manage ordering internally 
* `map` 
* `set` 
* `multimap` 
* `multiset` 
Unordered associative containers - implemented by hash table and hash table manage ordering, we can't change it:
* `unordered_map` (C++11)
* `unordered_set` (C++11)
* `unordered_multimap` (C++11)
* `unordered_multiset` (C++11) 
Container adaptors - essentially wrappers around some concrete containers to provide necessarily runtime properties:
* `queue` 
* `stack`
* `priority_queue`


### Iterators 
Iterators typically provide a way of observing a container's elements and ordering. Some Containers provide more than one way to observe elements.
Iterators may provide a way of modifying a container's elements.
An iterator's interface specifies 
* the complexity of observing and traversing a collection's elements
* the manner in which elements are observed 
* whether an element can be read from or write to 
**Iterators never own the elements to which they refer** 

Classic STL has five iterator categories: 
* Output / Input 
* Forward 
* Bidirectional 
* Random-access
This arranged in a hierarchy of requirements (not public inheritance): bidirectional iterator implements all the interface that required for forward iterator and some more staff (inheritance of interface, but not implementation). 


### Algorithms
The algorithms process ranges of elements of a collection. It require at least one explicitly-specified iterator pair.

Algorithm categories:
* Non-modifying algorithms - access elements without changing them or order;
* Modifying algorithms - can modify elements either emplace/copy or changing value of existing element;
* Removing algorithms - remove elements, actually working via rearranging elements moving them at the end of container where we don't see them anymore; 
* Mutating algorithms - can change order, but not values (reversing, rotating, shuffling etc);
* Sorting algorithms;
* Sorted range algorithms; 
* Numeric algorithms; 


## Iterators 
With iterators come to mind question:
* Where do the five iterator categories come from?
* What interface does each category provide? 
* What is their time complexity?
* How are they used by the algorithms? 

Let's try a generic programming exercise and develop iterators from scratch.
Back in the 90s C++ doesn't had templates etc, but had C.

Consider pointers to 2 elements in an array of N objects, `p` and `q`.
> $|e_0 | e_1 | ... | e_{p-1} | e_p | e_{p+1} | ... | e_{q-1} | e_q | e_{q+1} | ... | e_{N-1}|$ 

Here table what can we do with #raw_pointers (in C):

| Action                             | Operation                            |
| ---------------------------------- | ------------------------------------ |
| Access element                     | `*p`                                 |
| Access member of element           | `p->mem`                             |
| Compare for equality of position   | `p == q`, `p != q`                   |
| Move forward by 1                  | `++p`, `p++`                         |
| Move backward by 1                 | `--p`, `p--`                         |
| Make a copy (assgin)               | `q = p`                              |
| Access arbitrary element           | `p[n]`                               |
| Move forward by arbitrary n        | `p += n`, `q = p + n`                |
| Move backward by arbitrary n       | `p -= n`, `q = p - n`                |
| Compare for relative position      | `p < n`, `p <= q`, `p > q`, `p >= q` |
| Find distance between two elements | `d = q - p`                          |
An all this operations have O(1).

Consider pointers to 2 nodes in a simple doubly-linked list: 
> | next |->| next |->| next |->| next |->| next |->| next |->| null |
> | null |<-| prev |<-| prev |<-| prev |<-| prev |<-| prev |<-| prev |
> |  $e_f$  |   | ... |   |  $e_p$  |  |  ... |  |  $e_q$   |  | ...  |  |  $e_1$   |


| Action                           | Operation          |
| -------------------------------- | ------------------ |
| Access elements                  | `*p`               |
| Access member of element         | `p->mem`           |
| Compare for equality of position | `p == q`, `p != q` |
| Move forward by 1                | `p = p->next`      |
| Move backward by 1               | `p = p->prev`      |
| Make a copy (assign)             | `q = p`            |
An all this operations also have O(1).

Arrays, doubly-linked lists and singly-linked lists all supports multi-pass iteration. Pointers to elements can be dereferenced more than once, with the same result each time, the sequence can be iterated over (traversed) more than once. 

What about sequences that can be  traversed only once? Some sequences supports only single-pass iteration, like file streams/sockets/raw devices, etc. We can consider bytes coming out from stream or socket as sequence, but active iterating over it changes it. We can copy byte and dereference it, but we can't dereference random location behind socket. An element can only be read from, or written to, a given position one time, the act of reading or writing irrevocably changes position.

Consider a pointer to a FILE stream opened for input. Here traditional C structure and pointer to that black-box structure, not the bytes in the file. 
Reading:
`FILE* p = fopen(name, "rb");`
> p -> | $b_n$ | $b_{n-1}$ | $b_{n-2}$ | $b_{n-3}$ | ... |

| Action                           | Operation             |
| -------------------------------- | --------------------- |
| Read element and advance         | `b = fgetc(p)`        |
| Compare for end-of-file equality | `b == EOF`, `feof(p)` |
| Make a copy (assign)             | `q = p`               |
Writing:
`FILE* p = fopen(name, "wb");`
> p -> | $b_n$ | $b_{n-1}$ | $b_{n-2}$ | $b_{n-3}$ | ... |

| Action                           | Operation             |
| -------------------------------- | --------------------- |
| Write element and advance        | `fputc(b, p)`         |
| Make a copy (assign)             | `q = p`               |
All that also O(1) complexity 

Examples above makes iterators categories more clear: 
![[res/Pasted image 20241009134044.png]]

| Category      | Operation                               |
| ------------- | --------------------------------------- |
| Output        | Write forward, single-pass              |
| Input         | Read forward, singe-pass                |
| Forward       | Access forward, multi-pass              |
| Bidirectional | Access forward and backward, multi-pass |
| Random Access | Access arbitrary position, multi-pass   |
Arranged in a hierarchy of requirements. Not public inheritance, arrow to 'X' means "satisfies at least the requirements of X", dotted arrow means "optional".
Iterators that satisfy the requirements of output iterators are called mutable iterators. 


Let's think about sequences in terms of positions. By fiat, a sequence of N elements has N+1 positions. The first N positions contain elements and are dereferenceable. Assume the last position contains nothing and is therefore non-dereferenceable. We can point/refer to the last positions, but we cannot read from it or write to it. 

In the STL, iteration over sequences is based on the idea of _iterator ranges_, a pair of half-open interval `[first, last)` - `first` refers to the first element included in the sequence, `last` refence to the non-referenceable "one-past-the-end" (PTE) position excluded from the sequence. Half-open intervals makes testing for loop termination very simple. Loop only need to test for integrator equality to determine where loop should be terminated, there is no indexes involved (therefor random access not an issue) and location in the memory of the thing we iterator refers to is irrelevant. 
The way how iterator work depends on the container / sequence: 
* Containers that store elements contiguously in memory rely on ability to get a pointer to the "next-position-after". This guarantied by standard. 
* Node-based containers can use _sentinel nodes_, which represents that memory location passed last node.![[res/Pasted image 20241009150031.png]]

### Output Iterators 
It's write forward, single-pass iterator 
Standard guarantees following capabilities: 

| Expression  | Action/Result                                   |
| ----------- | ----------------------------------------------- |
| `Iter q(p)` | Copy construction                               |
| `q = p`     | Copy assignment                                 |
| `*p`        | Write to position by dereference, only one time |
| `++p`       | Step forward, return new position               |
| `p++`       | Step forward, return old position               |
The only valid use of the `*p` is on the left side of an assignment statement. 
Comparison operators are not required - no end of sequence is assumed ("infinite sink")
`const_iterator` types provided by STL containers cannot be output iterators - `const_iterator` permit only reading. 

### Input Iterators 
It's read forward, single-pass iterator 

| Expression  | Action/Result                              |
| ----------- | ------------------------------------------ |
| `Iter q(p)` | Copy construction                          |
| `q = p`     | Copy assignment                            |
| `*p`        | Read access to element one time            |
| `p->mem`    | Read access member of the element one time |
| `++p`       | Step forward, return new position          |
| `p++`       | Step forward, return old position          |
| `p == q`    | Return true if two iterators are equal     |
| `p != q`    | Return true if two iterators are different |
Two input iterators that are not pass the end iterators cannot compare equal if they refer to different positions. `p == q` doesn't imply `++p == ++q`, so it doesn't guarantee that after incrementing they will be at the same place. The comparison operators are provided to check whether an input iterator is equal to the past-the-end iterator. 
All iterators that read values must provide at least the capabilities of input iterators; usually, they provide more. 

### Forward iterator 
Access forward, multi-pass

| Expression  | Action/Result                                             |
| ----------- | --------------------------------------------------------- |
| `Iter q(p)` | Copy construction                                         |
| `q = p`     | Copy assignment                                           |
| `*p`        | Access element                                            |
| `p->mem`    | Access member of element                                  |
| `++p`       | Move forward, return new position                         |
| `p++`       | Move forward, return old position                         |
| `p == q`    | Return true if two iterators refer to the same position   |
| `p != q`    | Return true if two iterators refer to different positions |
| `Iter p`    | Default constructor, create singular value                |
`p` and `q` refer to the same position if `p == q`, that also mean `p == q` implies `++p == ++q`. Accessing an element (`*p`) doesn't change the iterator's position. 

### Bidirectional iterator 
Access forward/backward, multi-pass

| Expression  | Action/Result                                             |
| ----------- | --------------------------------------------------------- |
| `Iter q(p)` | Copy construction                                         |
| `q = p`     | Copy assignment                                           |
| `*p`        | Access element                                            |
| `p->mem`    | Access member of element                                  |
| `++p`       | Move forward, return new position                         |
| `p++`       | Move forward, return old position                         |
| `p == q`    | Return true if two iterators refer to the same position   |
| `p != q`    | Return true if two iterators refer to different positions |
| `Iter p`    | Default constructor, create singular value                |
| `--p`       | Move backward by 1, return new position                   |
| `p--`       | Move backward by 1, return old position                   |
Additional capabilities and guarantees: 
`p  == q` implies `--p == --q`; `--(++p) = p`.

### Random access 
Arbitrary access, multi-pass

| Expression       | Action/Result                                                   |
| ---------------- | --------------------------------------------------------------- |
| `Iter q(p)`      | Copy construction                                               |
| `q = p`          | Copy assignment                                                 |
| `*p`             | Access element                                                  |
| `p->mem`         | Access member of element                                        |
| `++p`            | Move forward, return new position                               |
| `p++`            | Move forward, return old position                               |
| `p == q`         | Return true if two iterators refer to the same position         |
| `p != q`         | Return true if two iterators refer to different positions       |
| `Iter p`         | Default constructor, create singular value                      |
| `--p`            | Move backward by 1, return new position                         |
| `p--`            | Move backward by 1, return old position                         |
| `p[n]`           | Access element at nth position                                  |
| `p += n`         | Move forward by n elements (and backward if n < 0)              |
| `p -= n`         | Move backward by n elements (and forward if n < 0)              |
| `p + n`, `n + p` | Return iterator pointing n elements forward (backward if n < 0) |
| `p - n`          | Return iterator pointing n element backward (forward if n < 0)  |
| `p - q`          | Return distance between positions                               |
| `p < q`          | True if p is before q in the sequence                           |
| `p <= q`         | True if p is not after q in the sequence                        |
| `p > q`          | True if p is after q in the sequence                            |
| `p >= q`         | True if p is not before q in the sequence                       |

Emulate raw pointers, provide operators for iterator arithmetic, analogous to pointer arithmetic, relation operators to compare positions (the element that p points to precedes or succeeds the element that q points to).


### Iterator Adaptors 
Iterator adaptors provide additional functionality beyond iterators themselves. 

* Reverse iterator: wraps container iterator like`template<class Iter> reverse_iterator`. Iterates backward from the end of a sequence to the beginning, working for bidirectional and random-access iterators. 
* Insert iterators (inserters): models an output iterator that inserts element at the back / front / interior of a container 
	* `template <class Container> back_insert_iterator`
	* `template <class Container> front_insert_iterator`
	* `template <class Container> insert_iterator`

## Containers 
_Sequence containers_ represent ordered collections where an element's positions is independent of its value. Usually implemented using arrays of linked lists. `vector, deque, list, array, forward_list`. Important thing is that the element's position is independent of its value. 

_Associative containers_: represent sorted collections where an element's position depends only on its value. Require from user some sort of strict weak ordering comparison operation (so elements can be sorted). Usually implemented using binary search trees (red black trees). `map, set, multimap, multiset`. 

_Unordered associative containers_: represent unsorted collections where an element's position is irrelevant. Implemented using hash tables. `unordered_map, unordered_set, unirdered_multimap, unordered_multiset`. 

Every STL container provides a common set of nested type aliases (there always nested public typedef iterator which allows to know what the type of iterator is that goes with this container in a generic fashion)
```c++
template< ... > 
class container 
{
	...
	using value_type      = ...
	using reference       = ...
	using const_reference = ...
	
	using iterator        = ... 
	using const_iterator  = ...
	using size_type       = ... // number of elements that container can hold
	using difference_type = ... // can reprecent distance between two iterators 
	...
}
```

Every STL container provides a common set of functions 
```c++
template< ... >
class container 
{
	...
	iterator       begin();
	iteraor        end();
	
	const_iterator begin() const;
	const_iterator end() const;

	const_iterator cbegin() const;
	const_iterator cend() const;
	...
}
```

Every bidirectional containers (and above) provide additional aliases and functions
```c++
template < ... > 
class bidirectional_container 
{
	...
	using reverse_iterator       = ...
	using const_reverse_iterator = ...

	reverse_iterator       rbegin();
	reverse_iterator       rend();

	const_reverse_iterator rbegin() const;
	const_reverse_iterator rend() const;
	
	const_reverse_iterator crbegin() const;
	const_reverse_iterator crend() const;
}
```

### Some containers overview
#### Sequence container _vector_
`template<class T, class Allocator = allocator<T>>`
`class vector;`
Features: 
* Supports amortized constant time insert and erase operations at its end 
* Supports linear time insert and erase operation in its middle 
* Provides const and mutable **random-access** iterators 
* Provides const and mutable element indexing
* Support changing element values 
* Uses contiguous storage for all element types except `bool`

#### Sequence container _deque_
`template<class T, class Allocator = allocator<T>>`
`class deque;`
Features: 
* Support amortized constant time insert and erase operations at the both ends
* Supports linear time insert and erase operations in its middle 
* Provides const and mutable **random-access** iterators 
* Provides const and mutable element indexing 
* support changing element values 
Deque don't have to store data contiguously. 

#### Sequence container _array_
`template<class T, size_t N>` 
`class array;`
Features: 
* Manages a fixed-sized sequence of objects in an internal C-style array
* Provides const and mutable **random-access** iterators 
* Provides const and mutable element indexing 
* Support changing element values
* Uses contiguous storage for all element types; 

#### Sequence container _list_
`template<class T, class Allocator = allocator<T>>`
`class list;`
Features: 
* Supports constant time insert and erase operations anywhere in the sequence 
* Provides const and mutable **bidirectional** iterators 
* Supports changing elements values 
* Provides member functions for splicing, sorting and merging 
* Usually implemented as a doubly-linked list 
Splice is an constant time operation

#### Sequence container _forward_list_
`template<class T, class Allocator = allocator<T>>` 
`class forward_list;`
Features: 
* Supports constant time insert and erase operations anywhere in the sequence 
* Provides const and mutable **forward** iterators 
* Supports changing elements values 
* Provides member functions for splicing
* Usually implemented as a singly-linked list 

#### Associative container _set_
`template<class Key, `
`        class Compare = less<Key>,`
`        class Allocator = allocator<Key>> `
`class set;`
Features: 
* Supports logarithmic time element lookup
* Elements of type `Key` are sorted according to `Compare`
* Elements keys are unique 
* Provides const **bidirectional** iterators 
* Usually implemented as a binary search tree 

_multiset_ - identical to set, but allow yo have non-unique keys

#### Associative container _map_
`template<class Key, class Val,`
`         class Compare = less<Key>,`
`         class Allocator = allocator<pair<const Key, val>>>`
`class map;`
Features: 
* Supports logarithmic time lookup of a type `Val` based on a type `Key`
* Elements of type `pair<const Key, Val>` are sorted according to `Compare`
* Keys values are unique 
* Provides const and mutable **bidirectional** iterators
	* Mutable iterators permit the `Val` member of `pair<const Key, Val>` to be modified
* Usually implemented as binary search tree 
* Can be used as an associative array

_multimap_ - same as map, but with unique keys. Except using as an associative array - can be used as a dictionary. 

#### Unordered associative container _unordered_set_
`template<class Key,`
`         class Hash = hash<Key>,`
`         class Pred = equal_to<Key>,`
`         class Allocator = allocator<Key>>`
`class unordered_set`
Features:
* Support amortized constant time element lookup
* Elements of type `Key` are stored internally in an order determined by `Hash`
* Element value are unique 
* Provides const **forward** iterators 
* Implemented as a has table - `Hash` helps determine ordering, `Pred` test `Key` equivalence 
When new element is added - no guarantee that any portion of the sequence will be the same as the iteration before. 

_unordeded_multiset_ - same as unordered_set, but with not unique values. 

#### Unordered associative container _unordered_map_
`template<class Key, class Value,`
`         class Hash = hash<Key>,`
`         class Pred = equal_to<Key>, `
`         class Allocator = allocator<Key>>`
`class unordered_map`
Features: 
* Supports amortized constant time lookup of a type `Val` based on a type `Key`
* Elements are of type `pair<const Key, Val>`
* `Key` values are unique 
* Provides const and mutable **forward** iterators 
* Implemented as a hash table - `Hash` helps determine ordering, `Pred` tests `Key` equivalence
* Can be used as associative array

`unordered_multimap` - same as unordered_map, but with non unique values. 

#### Container Adaptor _stack_
`template<class T, class Container = deque<T>>`
`class stack;`
Features: 
* Wrapper type that implements a classic LIFO stack
* Amortized constant time `push()` and `pop()` operations 
* Constant time access to next element with `top()`
* Works with `vecotr, deque, list, forward_list`
Requirements from `Container`
* Amortized constant time `push_back()` and `pop_back()` member functions 
* constant time `back()` member function 

#### Container Adaptor _queue_
`template<class T, class Container = deque<T>>`
`class queue;`
Features: 
* Wrapper type that implements a classic FIFO stack
* Amortized constant time `push()` and `pop()` operations
* Constant time access to next element with `front()` and last element with `back()`
* Works with `vector, deque, list and forward_list`
Requirements from `Container`
* Amortized constant time `push_back()` and `pop_back()` member functions 
* constant time `front()` and `back()` member function 

#### Container Adaptor _priotiry_queue_
`template<class T, class Container = deque<T>>`
`class priority_queue;`
Features: 
* Wrapper type that implements a classic priority queue (AKA heap)
* Logarithmic time `push()` and `pop()` operations
* Constant time access to next element with `top()` 
Requirements from `Container`
* Amortized constant time `push_back()` and `pop_back()` member functions 
* constant time `front()` member function 
* Random-access iterators (works with `vector` and `deque`)

## Algorithms 
There's large number of algorithms provided by STL (well over 100), multiple versions of almost all, some of them parallel.

Algorithm categories: 
* Non-modifying algorithms 
* Modifying algorithms 
* Removing algorithms 
* Mutation algorithms 
* Sorting algorithms 
* Sorted range algorithms 
* Numeric algorithms 

Some quick examples: 

_std::sort_
```c++
template<class RandmoIter, class Compare> 
void 
sort(RanomdIter first, RandomIter last, Compare comp)
```
Sorts the elements in the range `[first, last)` in non-descending order; the order of equivalent elements us not guaranteed to be preserved; Elements are compared using the given binary comparison function `comp`
Complexity: `O(N log(N))`, where `N  std::distance(first, last);
From function arguments we can notice, that it's require random-acces iterators. 

_std::lower_bound_
```c++
template<class ForwardIter, class T>
ForwardIt
lower_bound(ForwardIter first, ForwardIter last, const T& value);
```
Returns an iterator pointing to the first element i the range `[first, last)` that is not less than `value`, or `last` if no such element is found.
Complexity: the number of comparisons performed is logarithmic in the distance between `first` and `last` (at most log2(last-first) + O(1) comparisons)
For non-random-access iterators, the number of iterator increments is linear. 

_std::remove_copy_if_ 
```c++
template<class InputIter, class OutputIter, class UnaryPredicate>
OutputIt
remove_copy_if(InputIter first, InputIter last, OutputIter dest, UnaryPredicate pred)
{
	for (; first != last; ++first)
	{
		if (!pred(*first))
		{
			*dest++ = *first;
		}
	}
	return dest;
}
```
Copies elements from the range `[first, last)`, to another range beginning at `dest`, omitting the elements which sarisfy specific criteria; source and destination ranges cannot overlap; return an iterator to the element past the last element copied.
Complexity: exactly `std::distance(first, last)` applications of the predicate. 