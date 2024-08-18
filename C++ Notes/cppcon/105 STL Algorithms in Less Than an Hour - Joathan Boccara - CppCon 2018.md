## Permutations 
it's algorithm that rearange\move around elements without modifying them
### Heaps 
Heap - is tree structer where childrens always less then parents. 
```
                 /  [9]     \
           /   [8]  \    / [6] \ 
         /[7]\     /[4] [5]   [2] 
       [0]   [3]  [1]  
```
Heaps can be converted into array `[9][8][6][7][4][5][2][0][3][1]` and childrens can be access by multiplying index by 2. 
+ `std::make_heap(begin(numbers), end(numbers))` - meake from numbers array a heap
+ `std::push_heap(begin(numbers), end(numbers))` - add element into heap. First(?) add new element in back `numbers.push_back(8.88)` and then rebalance array in proper order. 
+ `std::pop_heap(begin(numbers), end(numbers))`  - get first element (which will be maximum element in that collection),  swap first and last elements and rebalanced heap. Then with `numbers.pop_back()` possible to remove element or if not remove and continue - we get sorted collection.
+ `std::sort_heap` - sort heap. 
### Sorting 
+ `std::sort` - regular collection sort.
+ `std::partial_sort` - sort from beggining till specified possition, rest will be in unsorted order. 
+ `std::nth_element` - point in nth possition element that would be there if whole collection was sorted and put on left - everting that is smaller and on the right - what is bigger (in unspecified order both) 
+ `std::inplace_merge` - take two sorted collections and merge them in one sorted collection? 
### Partitioning 
`std::partition` - look at set throught predicat (some boolean criteria) and put at the beggining everything that satisfy true evaluation and not satisfy after at the end. 
+ `std::partition_point` - boarder between two subsets, end of 'true' range and begging of 'false' range 
   `1  3  5` 2  4
           ^  - partition_point
### Other permutations
+ `std::rotate` - take last elemnt and put it in the front 
+ `std::shuffle` - rearange collaction in random order. 
+ `std::next_permutation` and `std::prev_permutation` - rearange to next\previous permutations
+ `std::reverse` - rearange in backward order 

## Secret runes 
Things, that can be combined with rest algorithms to generate new algorithm
### Partioning-sort-heap
+ `stable_*` (`std::stable_sort`, `std::stable_partition`) - keep relative order. 
+ `is_*` (`is_heap`, `is_sorted`) - returns boolean in case if algorithm will not make changes
+ `is_*_until` (`is_partitioned_until`) - return iterator that the first position where that pradicat doesn't hold anymore? 
+ `*_copy` (`std::remove_copy`, `rotate_copy`,`partial_sort_copy`) - do some algorithm but in new collection. 
+ `*_n` (`std::copy_n`, `std::destroy_n`, `std::search_n`) - take begin and size.
+ `*_if` (`find_if/find_if_not`, `remove_if`, `copy_if`) - as a predicate 

## Queries
Algorithms that doesn't change anything but more likely extracat value.
### Numeric algorithms 
+ `std::count` - return how many times some value accure in collection 
+ `std::accumulate` - returns sum of elements (or any custom function?) 
+ `std::reduce` - same as accumulate, but a little different interface, like 'no initial value?' and can run in parallel
+ `std::transform_reduce` - takes some function and apply before calling reduce. 
+ `std::partial_sum` - sums all elements from beggining to current point of collaction  returns collaction with result (`1th, 1th+2nd, 1th+2nd+3rd'..)
+  `std::(transform_)inclusive_scan/exclusive_scan` - `inclusive_scan` same as partial_sum but can run in parallel, `exclusive_scan` same as `inclusive_sum` but not take in sum current element, `transform_` - same as for `transform_reduce` - take function and apply before. 
+ `std::inner_product` - perform inner protuct (multiplication of counter part elements and sum of result)
+ `std::adjacent_difference` - return difference between two neighbors elements `(0-1, 1-2, 2-3, 3-4`) 
+ `std::sample` - takes amount `n` and return randomly selected `n` elements
### Querying a property
+ `std::all_of` - takes collection and predicat and returns true if all elements satisfy predicat. For empty collection return true. 
+ `std::any_of` - takes predicat and return true if at least one element satisfy predicat. With empty collection return false.
+ `std::none_of` - non of elements satisfy predicat. With empty collaction return true.

### Querying a property on 2 ranges
+ `std::equal` - if elements equal and have same size
+ `std::is_permutation` - if equal but not necessary in same order. 
+ `std::legicographical_compare` - return in form of boolean is first range is smaller if they where in the dictionary (abcdefgh smaller than abcdyz even if longer)
+ `std::mismatch` - return `std::pair<iterartor, iterator>` where ranges differ

### Searching a value 
#### For not sorted
+ `std::find` - return iterator where given element first accure
+ `std::adjacent_find` - return  for given value position where two of that value appeare in a row 
#### For sorted 
+ `std::equal_range` - search for subrange with given number
+ `std::lower_bound`/`std::upper_bound` - possition where begin/end of subrange (so to insert before range - use `std::lower_bound`  and after - `std::upper_bound`) 
```
                  |   | - std::equal_range
              1 2 3 3 3 4 5
 std::lower_bound |     |- std::upper_bound  
``` 

+ `std::binary_search` - takes collaction and value and return boolean is value in collaction or not. 

### Searching a ragne 
 + `std::search` - find range \[3 5 2\] in \[1 2 7 `3 5 2` 1 5 3 5 2\]`
 + `std::find_end` - same as search, but from end  \[1 2 7 3 5 2 1 5 `3 5 2`]`
+ `std::find_first_of` - find any first match from range \[5 2 3] in  \[1 `2` 7 3 5 2 1 5 3 5 2\]`

### Searching a relative value 
+ `std::max_element`/`std::min_element` - return iterator of max/min element in collaction
+ `std::minmax_element` - return max and min element as `std::pair<iterator, interator>`

## Algorithms on Sets 
Set - any sorted collaction
+ `std::set_defference (begin(a), end(a); begin(b), end(b), std::back_inserter(results))` -  return elements that in set A, but not in set B. And it's in linear complexity
+ `std::set_intersection` - returns elements that in both sets.
+ `std::set_union` - return all elements from A and B
+ `std::set_symmetric_difference` - return elements that not in both sets. 
+ `std::includes` - returns a boolean is set A fully include set B
+ `std::merge` - put two sets in one big set. 

## Movers
Move ranges around. 
+ `std::copy(first, last, out` - copy one collaction into out
+ `std::move(first, last, one` - not same as `std::move(val)`, it's works as `std::copy` but calls usual `std::move` for each element. 
+ `std::swap_ranges(first, last, out)` - swap collactions of same size
+ `std::copy_backward`/`std::move_backward` - copy first range \[1 2 3 4 5 6 7 8 9 10\] in such way that result will be \[1 2 3 1 2 3 4 5 9 10\]

## Value modifiers
they can change data inside collaction
+ `std::fill(first, last, value)` - puts value in every element of collaction 
+ `std::generata(first, last, f)` - call function (without arguments) for every element of collaction 
+ `std::iota(first, last, value)` - put value at the beggining and call val++ for every next element.
+ `std::replace(first, last, exist_value, new_value)` - well, replace what exist_value in collaction with new_value.

## Structure changers
note that algorithms operate with iterators, thus they can't changes size
+ `std::remove(begin, end, value)` - remove given value existing in collaction and pull other value on the left, leaving on the right not specified values and return iterator where end of valid area. (So, `collection.erase(std::remove(begin(collection), end(collection), value), end(collection))` -  shrink als invalid right area. 
+ `std::unique(begin(collection), end(collection))` - remove adjacent values (so, \[1 3 99 99 99 4 99\] become \[1 3 99 4 99\])

## Transform
+ `std::transform(begin(colection), end(collection), std::back_inserter(results), f)` - apply some function on each elent and output result into new collections
    \[x1, x2, x3, x4\] -> f -> \[f(x1), f(x2), f(x3), f(x4)\]
* `std::transform(begin(colection), end(collection), begin(ys), std::back_inserter(results), f)` - same but feed in function 2 collections: 
	\[x1, s2, x3, x4\]  \\
	              -> f -> \[f(x1, y1), f(x2, y2), f(x3, y3), f(x4, y4)\]
    \[y1, y2, y3, y4\] / 
## for_each 
`std::for_each(begin(collection), end(collection), f)` - almost similar to transform, but doesn't care about result, simply calls with function f each element of collaction. Since function return void -  there can be side effects.


## Raw memory 
+ `std::fill`, `std::copy`, `std::move` - they are use `operator=`. But in some cases we have object that not constructed yet, just have junk of memory and what to call construct instead assignment operator.  For this used `std::uninitialized_fill`, `std::uninitialized_copy`, `std::uninitialized_move`
+ `std::destroy` - call destructor 
+ `std::``uuuninitilized_default_construct`/`std::uninitilized_value_construct` - call default/value contructor