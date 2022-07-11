---
title:  "Sort Algorithms Cheatsheet"
date:   2020-04-09 09:00:00
permalink: sort-algorithms-cheatsheet
---

This is a summary of the key features that make up an algorithm.

## Motivation

While it is easy to understand the concept of each sort algorithm, I find it difficult for me to remember the key steps that define an algorithm or is characteristic of that algorithm. That key step may prove to be pivotal in solving the algorithm but may come in as insignificant from the macro view of its concept.

Hence, I decided that it will be better for me to jot the key pointers down.

## Quicksort

```ruby
def quicksort array
  recursion(array, 0, array.length - 1) #IMPT
  array
end

def recursion array, start, finish
  if start < finish # IMPT
    pivot_index = partition(array, start, finish)
    recursion(array, start, pivot_index - 1) # IMPT
    recursion(array, pivot_index + 1, finish) # IMPT
  end
end

def partition array, start, finish
  pivot = array[finish]
  pivot_index = start

  (start...finish).each do |index| # IMPT
    if array[index] <= pivot # NOTE
      array[index], array[pivot_index] =
        array[pivot_index], array[index]
      pivot_index += 1
    end
  end

  array[finish], array[pivot_index] =
    array[pivot_index], array[finish]

  pivot_index
end
```

### Key Steps

Gist: Using a pivot value, distribute the array into 2 halves that are not ordered, but are collectively smaller on the left side and collectively larger on the right side.

The array is mutated.

The pivot value of each iteration will find its rightful position in the array at every iteration, eventually leading to a sorted array.

### Discussions

Let’s start with the recursion function.

In the recursion function, note that the arguments are indices of the array, not the length. Keep this at the back of your mind so that you can understand when to end a loop.

Line 7 ensures we are iterating at least 2 elements.

In lines 9 and 10, the recursion occurs on either side of the pivot index in that iteration. Note that the pivot index does not participate in the next recursion, since it is already at where it belongs.

Now for the partition function.

In the partition function, the pivot does not participate in the reordering. Line 18 ensures the loop ends before reaching the last index, finish, which is the pivot, with the non-inclusive range constructor operator.

In the loop in line 18, we are are trying to push the values smaller than or equal to the pivot to the left here. It is also ok to use <.

We do so by swapping them with those that are bigger than the pivot, but exist on the left of those that are smaller.

The pivot_index increments at each swap and remembers the last position that was swapped. Hence, at the end of the loop, it holds the position of the first value that is bigger than the pivot. Everything on the left is either smaller than or equal to the pivot.

This is where the pivot belongs to in the array. We swap the pivot into that position. Ascend the throne!

The state of the array does not change in this last swap: all elements on the left of the pivot is still smaller or equal to the pivot, while all elements on the right of the pivot is still bigger than the pivot. They remain unsorted

The function returns the pivot’s position to the parent recursion function, which needs it to know where to split the array for the next iteration.

Lastly, let’s go back to the calling function where the initial recursion function is triggered. Make sure to pass in the last index of the array instead of its length.

## Mergesort

```ruby
def merge_sort(list)
  return list if list.length <= 1

  mid = list.length / 2
  left = merge_sort(list[0...mid])
  right = merge_sort(list[mid...list.length])
  merge(left, right)
end

def merge(left, right)
  return right if left.empty?

  return left if right.empty?

  if left.first <= right.first
    [left.first] + merge(left[1...left.length], right)
  else
    [right.first] + merge(left, right[1...right.length])
  end
end
```

### Key Steps

Gist: first recursively halve array until we are dealing with 1 element, then recursively merge the elements back in a sorted order until we get back the array of the same size, and now it will be sorted

A recursive function that consist of 2 parts in order: recursively split and recursively merge

The array will be mutated

In lines 16 and 18, we are continuously appending the smaller of the first element on the right vs left array.

Lines 11 and 13 will take care of the comparison that is still ongoing, when 1 side has been fully appended while the other still have elements inside. Since these arrays are already sorted at whichever iteration, we can just append the whole array.

Remember the breaking function in line 2

Unfortunately, while this use of recursion is great, the number of recursions may become too excessive and cause a “stack level too deep” error.

We may need to to think prepare an alternative if the stack overflows.

```ruby
def merge_sort(list)
  return list if list.length <= 1

  mid = list.length / 2
  left = merge_sort(list[0...mid])
  right = merge_sort(list[mid...list.length])
  merge(list, left, right)
end

def merge(array, left, right)
  left_index = 0
  right_index = 0
  index = 0

  while left_index < left.length &&
    right_index < right.length
    if left[left_index] <= right[right_index]
      array[index] = left[left_index]
      left_index += 1
    else
      array[index] = right[right_index]
      right_index += 1
    end
    index += 1
  end

  array[index...index + left.length - left_index] =
    left[left_index...left.length]
  array[index...index + right.length - right_index] =
    right[right_index...right.length]

  array
end
```

Line 15 till 36 basically carry out the operation with a while loop instead of recursion. It mutates the array along the way.

## Insertion sort

### Key Steps

Gist: insert elements one by one from unsorted part of array into sorted part of array

Divide the array into sorted portion and unsorted portion

Sorted partition always starts from the first element, as array of 1 element is always sorted

First element of unsorted array will shift forward until the start of the sorted portion of the array OR until it meets an element bigger than itself

Order of the sorted portion is maintained

The last element of the sorted array takes its place

The next iteration start on the next element of the unsorted portion, which is now the first element of the current unsorted portion

The loop mutates the array

### Discussions

Best case is an already sorted array, so no shifting of elements from the unsorted to the sorted portion of the array, resulting in a time complexity of n

The worst case is a reverse sorted array, which results in the whole sorted array having to shift for each iteration. The first element of the unsorted portion of array is always at the the smallest and need to go to the front of the sorted portion. Time complexity is n^2

## Selection sort

### Key Steps

Gist: scan array to find the smallest element and eliminate it for the next iterations

Swap smallest element with the front most element

Scan the array in the next iteration excluding the smallest element(s)

Last remaining single element will be of the largest value, so iterations take place until n - 2

### Discussions

Time complexity is n^2

## Bubble sort

```ruby
def bubble_swap array
  swap_took_place = true
  while swap_took_place
    swap_took_place = false
    (0...array.length - 1).each do |index|
      if array[index] > array[index + 1]
          array[index + 1], array[index] =
            array[index], array[index + 1]
          # increment swaps here to record
          # number of swaps that took place
          swap_took_place = true
      end
    end
  end
  array
end
```

### Key Steps

Gist: keep swapping adjacent elements if left is larger than right down the array, and repeat this iteration for as many times as there are elements in the array. The last iteration will not have any swap occur to declare the array swapped.

### Discussions

Time complexity is n^2
