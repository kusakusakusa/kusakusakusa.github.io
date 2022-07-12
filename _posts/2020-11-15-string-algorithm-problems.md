---
title:  "String Algorithm Problems"
date:   2020-11-15 09:00:00
permalink: string-algorithm-problems
---

## Motivation

I tend to forget the flow of an algorithm and the reasons for executing certain steps that may be pivotal in getting the correct algorithm. So as a revision and as a post to my future self, here are my notes for the common algorithms related to strings.

## Longest Common Subsequence

There’s a number of ways to solve this. First is using recursion, which has a caveat but forms the core to the optimized solution using dynamic programming. Let’s take a look at solving for longest common subsequence problems using recursion.

### Solving Longest Common Subsequence problems using Recursion

```ruby
def lcs(s1, s2)
  define_method :recursion do |index1, index2|
    case true
    when index1 == s1.length || index2 == s2.length
      return 0
    when s1[index1] == s2[index2]
      return 1 + recursion(index1 + 1, index2 + 1)
    else
      return [
        recursion(index1 + 1, index2),
        recursion(index1, index2 + 1)
      ].max
    end
  end

  recursion(0, 0)
end

lcs(s1,s2)
```

Note: the define_method is a ruby method commonly used in dynamic programming in ruby. I’m using it lazily here to allow the nested recursion method to be able to access the variables s1 and s2 in the parent scope. It is not necessary to use it and there are other ways to construct this.

The idea here is to constantly compare each character in s1 with each character in s2 to accumulate the count if their characters match, and move on to the next character along each string.

The breaking function, which will cause the recursion algorithm to stop, is when we reach the end of either string. In that scenario, the count will not be incremented.

The caveat here is that there is a repetition of computation of the characters of either string as we progress through the other.

Some memoization will help with this longest common subsequence problem. We do that with a table.

### Solving Longest Common Subsequence problems using Dynamic Programming and Memoization

```ruby
def lcs(s1, s2)
  table = Array.new(s1.length + 1) {
    Array.new(s2.length + 1) {0}
  }

  (1...s1.length + 1).each do |s1i|
    (1...s2.length + 1).each do |s2i|
      if s1[s1i - 1] == s2[s2i - 1] # IMPT
        table[s1i][s2i] =
          table[s1i - 1][s2i - 1] + 1 # IMPT
      else
        table[s1i][s2i] = [
          table[s1i - 1][s2i],
          table[s1i][s2i - 1]
        ].max
      end
    end
  end

  table.last.last
end
```

The approach here is to construct a table that will memorize the steps calculated before. This is the common approach to LCS questions and I believe the video below best explain the theory behind this.

https://youtu.be/sSno9rV8Rhg

---

Some things to note.

- The last element of the table will accumulate the length of common subsequence, so looking at the last element in the table will get you the maximum.
- In line 6, make sure to minus 1 off the index when getting the character in the string. This is because the table has 1 more element than the string along its corresponding dimension, so we need remove the offset.
- Take note of the order of index to call in the multidimensional table. I tend to mix them. We are traversing vertically down the table first, then horizontally, and accessing the element is index from the vertical first then the horizontal, which is, unfortunately, opposite of the coordinate system where the horizontal (x) comes before the vertical (y).

---

To get the subsequence (not the length), you will need to trace back from the table, as explained in the video. Here is the code for that.

```ruby
def lcs s1, s2
  ... # after getting the table

  i = s1.length
  j = s2.length
  string = ''

  while table[i][j] != 0
    value = table[i][j]

    case true
    when value == table[i][j-1]
      j -= 1
    when value == table[i-1][j]
      i -= 1
    else
      string += s2[j - 1] # NOTE
      i -= 1
      j -= 1
    end
  end

  string.reverse
end
```

We start off at the last element (bottom right cell) of the table, where i and j is the length of each string respectively.

Remember, the table has 1 more element on each dimension than its respective string, so assigning each string’s length to i and j respectively will point to the last element in the table.

In line 8, we start the loop of backtracing.

The breaking condition is to check if the current element in the table is 0 or not. If it is zero, it means all possible matching characters have been accounted for and there are no more matching character moving forward along to the front of either string. We do not need to worry about going out of index also, because the 1st horizontal and vertical of the table contains 0, so the backtrace will cease when it reaches there under this same breaking condition.

Line 12 to 15, we check if the current value in the table is derived from the table element on top of it or on its left. The order does not matter, as either way will bring us to the character that is eventually matching on both string at their respective correct position.

In line 17 to 19, the value in the current table element does not have the same value as either its top and left elements. This means there was a match and it got its value from the diagonal element. Remember, when there was a match, the table element will take the diagonally top left element and add 1 to itself.

On line 17, note that you can use either character from either string, since they are matched. Just make sure you are using the correct index for the correct string. Remember to offset 1 from the index as the table has 1 more element that the string in the corresponding dimension.

We append the matching character to a string and continue to loop. Eventually the string will contain the matching characters and we just have to reverse it to get the longest common subsequence in the correct order.

---

However, if you just need the length of the longest common subsequence, you can do some space optimization by using 2 rows instead of a table. Afterall, the only rows involved in the computation is the current and the previous row.

The idea here is to set the current_row as the previous_row once the row has completed its round of iteration, because the current_row will become the previous_row in the next iteration.

```ruby
def lcs s1, s2
  previous_row = Array.new(s1.length + 1) {0}
  current_row = Array.new(s1.length + 1) {0}
  
  row_index = 1
  while row_index < s2.length + 1
    (1...current_row.length).each do |col_index|
      if s1[col_index - 1] == s2[row_index - 1] # IMPT
        current_row[col_index] =
          previous_row[col_index - 1] + 1
      else
        current_row[col_index] = [
          previous_row[col_index],
          current_row[col_index - 1]
        ].max
      end
    end

    previous_row, current_row =
      current_row, previous_row
    row_index += 1
  end
    
  previous_row.last
end
```

With that in mind, we can further optimize by observing the pattern that all the elements, starting from the front, in the previous_row will have the same value as the elements in the current_row until the element where there is a match. From there on, the same calculation resumes.

If there is no match, the whole current_row will have the same values as the previous_row.

Hence, we can skip the loop for these elements that we know will stay the same.

```ruby
def lcs s1, s2
  previous_row = Array.new(s1.length + 1) {0}
  current_row = Array.new(previous_row.length) {0}
  
  row_index = 1
  while row_index < s2.length + 1
    s2char = s2[row_index - 1]
    matched_index = s1.index(s2char)

    if matched_index.nil?
      current_row = previous_row.dup # IMPT
    else
      col_index = matched_index + 1
      current_row[0...col_index] =
        previous_row[0...col_index].dup # IMPT

      while col_index < current_row.length
        if s1[col_index - 1] == s2char
          current_row[col_index] =
            previous_row[col_index - 1] + 1
        else
          current_row[col_index] = [
            previous_row[col_index],
            current_row[col_index - 1]
          ].max
        end
        col_index += 1
      end

      previous_row, current_row =
        current_row, previous_row
    end
    
    row_index += 1
  end
    
  previous_row.last
end
```

Here are some differences with the previous algorithm.

Line 8 will check if there is any character in s1 that matches the current character of s2 that we are looking at.

If there are no matches, the previous_row‘s value is copied over to the current_row. Be sure to use dup function to make sure it is a deep copy so that we are not manipulating the same object.

If there is a match, matched_index will hold the index of the first matched character in s1. We have to add 1 to it as the current_row is set to be longer than s1 by 1 (check out the video once again if you don’t understand why).

From there, we will follow the same logic of populating the rest of the current_row.

Note that we can only do the matching index trick for the first character that matches. The rest of the row will need to be recalculated once the change comes in.

## Longest Common Substring

A similar approach to solving longest common subsequence solves the problem of Longest Common Substring. The difference between longest common substring and longest common subsequence is that the former must be a contiguous set of characters in the string, while the latter need not be contiguous as long it is sequential.

By the way, “contiguous” means elements in being adjacent to one another, while continuous means never ending.

It also involves a similar table, and we will see how it differs in the algorithm below.

```ruby
def lcs(s1, s2)
  table = Array.new(s1.length + 1) {
    Array.new(s2.length + 1) {0}
  }

  max = 0
  indices = []
  (1...s1.length + 1).each do |s1i|
    (1...s2.length + 1).each do |s2i|
      if s1[s1i - 1] == s2[s2i - 1] # IMPT
        table[s1i][s2i] =
          table[s1i - 1][s2i - 1] + 1 # IMPT

        # this part onwards depend on the question
        if table[s1i][s2i] > max
          indices = [[s1i, s2i]] # IMPT
          max = table[s1i][s2i]
        elsif table[s1i][s2i] == max
          indices << [s1i, s2i]
        end
      end
    end
  end

  # do something with max or indices or table
end
```

Relative to the operations required for Longest Common Subsequence, this is much simpler.

In line 2, we create the same kind of table, where it’s longer than the corresponding string along the corresponding dimension by 1, and it is filled up by 0.

We then loop each element in the table, ignoring the first row and column.

We check whether the corresponding characters at each string at the corresponding indices match. Take note to minus 1 from the corresponding index when accessing the character.

If they do not match, we set that element as 0, and since the table is already filled with 0, we do not need to do anything.

Now that the else condition is out of the way, let see what we do when there is a match.

We simply add 1 to the value at the element at the diagonally top left of the current element, as shown in lines 11 and 12. This is essentially the formula.

The other steps at line 15 to 20 will be dependent on what your question is. In this case, I am keeping track of the largest common substrings and taking note of their indices.

Line 19 appends the current indices when there is a tie on the longest substring length, while line 16 nullifies the old records and replaces it with the current indices as there is a new maximum. Note that we are assigning it an array of array.

What’s left is how you want to use the data after the computation. The maximum length of the common substring can be easily obtained from the max variable.

What about to find the longest substring(s)?

```ruby
def lcs(s1, s2)
  ... # after getting the table and indices
  strings = []
  indices.each do |index_pair|
    index1 = index_pair[0]
    index2 = index_pair[1]
    string = ''
    while table[index1][index2] != 0
      string += s1[index1 - 1] # IMPT
      index1 -= 1
      index2 -= 1
    end
    strings << string.reverse
  end

  strings.uniq
end
```

We continue from where we left off. We loop through the indices that hold the end positions of the longest common substrings between the 2 strings. These common substrings can be the same, and can also be different.

So for each pair of index in each element on indices, we add the character at the position, and move on to the element in its diagonally top left position. We append the characters as long as the value at its position in the table is not 1, signifying a match.

2 things to note when recording the character:

- Remember to minus 1 from the index because the table’s dimension is longer that the corresponding string
- While you can use either string to get the character since they will be the same as there is a match, you need to remember to use the correct index to access the correct string.

The resultant string is the common substring, but in reverse. Make sure to reverse it as shown in line 13.

As there may be repeated common substrings, use uniq like in line 16 to remove duplicates as required.

## Z Algorithm

There are many pattern matching/string search algorithm developed over the years. Examples include [Knuth-Morris-Pratt algorithm](https://en.wikipedia.org/wiki/Knuth%E2%80%93Morris%E2%80%93Pratt_algorithm) and [Rabin-Karp algorithm](https://en.wikipedia.org/wiki/Rabin%E2%80%93Karp_algorithm), [Aho-Corasick algorithm](https://en.wikipedia.org/wiki/Aho%E2%80%93Corasick_algorithm) and [Boyer-Moore string-search algorithm](https://en.wikipedia.org/wiki/Boyer%E2%80%93Moore_string-search_algorithm).

Z algorithm is another example and we will look into its implementation.

This video clearly explains the mechanism andI give it full credit. In this article, I will just be highlighting the key points based on the mistakes that I usually made while constructing the algorithm without reference.

https://youtu.be/CpZh4eF8QBw

Here’s a simple summary and refresher about what z algorithm is.

Z algorithm uses a “z-box” to mark characters whose neighboring patterns have been calculated before and save on those calculations and thus the iterations through them for optimization sake.

The steps involve generating this legendary z-box and

One caveat of the “z-box” occurs where the number subsequent matching characters reaches the end of the z-box. We cannot confidently say that the next character beyond the right end of the z-box does not contribute to a longer pattern match, so we need to take note of how we mark the end of the z-box.

With that, let’s take a look at the algorithm.

def zalgorithm s1, s2
  l = r = index = 0
  s = "#{s1}|#{s2}"
  z = Array.new(s.length) {0}
  index = 1

  while index < s.length
    if index > r
      l = r = index
      while r < s.length && s[r - l] == s[r]
        r += 1
      end
      z[index] = r - l
      r -= 1
    else
      if z[index - l] > r - index # IMPT
        l = index
        while r < s.length && s[r - l] == s[r]
          r += 1
        end
        z[index] = r - l
        r -= 1
      else
        z[index] = z[index - l]
      end
    end

    index += 1
  end

  z
end
We setup the variables required for the loop in lines 2 to 5.

l and r marks the left and right edge of the z-box that can be generated multiple times in the loop.

We combine s1 and s2 separated by a | character, assuming this | character will not appear in either string. We are trying to match s1 pattern where it appear in s2.

z in line 4 is not the z-box. It holds the number of subsequent subsequent that matches the pattern that we are looking for.

Lastly, we set the index to start at 1 because the first character is not matching with anything. And yes, we are starting, possibly, from within s1 to record any repeated pattern within s1 itself.

Now for the loop. It consist of an if-else statement at the top level.

In line 9 to 14, this logic is executed when we are out of the current z-box. We are no longer able to predict the number of matching characters from now on, so it is time to form a new box, and lines 9 to 14 does exactly this.

We set l to match the current index of the current iteration, and we want to find the end of this z-box that we are creating, which means to give r a value.

In line 10, we take care not to exceed the length of the string itself while finding the end of the z-box. We are also matching whether this current character matches the character in s1 at the same order (r - l gives the position at the start of the string, which is the pattern we are trying to match). This will also trigger a stop when we hit the special | character and incorrectly take s2 into account as part of the pattern to match.

We then record down how many of the subsequent characters from the current character matches the pattern, with the same order. Then come the important step at line 14. We do not include the last element that r currently holds as part of the z-box because there is a possibility of the pattern to continue matching with the character(s) beyond the end of the z-box. Taking it out of calculation will prevent this tragedy.

We now take a look at what the algorithm does when it has a z-box and the current iteration is inside it. Line 16 to 25 handles this.

We need to check if the recorded number of repeated string will reach the end of the z-box.

In line 16, index - l is the position of the pattern where the current character is expected to match, and we check its position in z to see if how many characters after that matches it. If this values bring us to beyond the edge of the z-box, we need to regenerate the z-box starting from the current character. Remember, we took out the last element of the z-box in line 14, so the comparator operator here will just be >.

Otherwise, we can simply copy the corresponding values that we recorded initially at the corresponding position, as depicted at line 24.

Regenerating the z-box is done in line 17 to 22 is exactly the same process as the case when we are outside the z-box, with only line 17 being different.

We only set l to be equal to index while r remains at its position, marking the edge of the z-box. This is because we are continuing to expand the z-box and predict the matching patterns of the subsequent characters beyond the edge of the previous z-box. We do not need to repeat what we have calculated from index to r.

Eventually, the z array will contain the number of subsequent characters matching from each index. We can manipulate the values in the z array to achieve what we need, like where are the positions of the full matches, how many full matches were there, etc.

## Things to note

You can apply z algorithm on a single string, for the case of finding the suffixes that match the prefix of the string. In this case, you do not need to add the special character | as we did in line 3.

z algorithms find matches to the pattern at the start of the string. It does NOT match subpattern, that is the matching patterns start from anywhere in the pattern but the first character.

## Window Sliding Algorithm for Substring problems

The window sliding technique is useful for problems involving consecutive elements. The object is usually to derive some results from some of these substrings, whether it is the minimum or maximum length of the substring that passes certain criteria, or whether substring contains certain characters.

Since this applies to consecutive elements, note that the window sliding technique can also be applied to non string problems, probably involving arrays or even linked lists.

The signature of a window sliding solution looks like this.

```ruby
def solution args
  # setup variables
  
  window_end = 0 # change start point
  while window_end < array.length
    # deal with the current element first

    # check if certain condition is maintained
    # if not, adjust the start of the window to fulfill maintain the criteria

    window_end += 1
  end

  # return ans
end
```

The algorithm will involve looping the array to make sure we touch on every element. As we loop though the array, we shift the window along (marked by the ever increasing window_end variable that marks the end of the window) and carry out our logic to find our answer.

Once again, this sliding window method works exactly because we are looking at CONSECUTIVE elements. This property allows us achieve our answer within 1 pass.

At the start of the loop at line 6, we always deal with the current element first. Whether it is appending it to some array that we will need to keep track of, or accumulating its value for some calculation related to the answer, we deal with it first. This puts the current element into the sliding window for analysis, taking into account its effect on the sliding window.

Line 8 and 9 are some comments. They mark the location where the main analysis of the window take place. Here, depending on the question, it may consist of a while loop (yes, an inner while loop) to ascertain the start position of the window, usually when a criteria is not fulfilled.

This inner while loop will continuously shift the start of the window towards the right (the right end of the window is in standstill at this moment) as it seeks out the new start of the window where the criteria is fulfilled.

Now, we will need to track the start of the window, so a window_start variable is required. Remember to increment this window_start variable. With some arithmetic operations with the window_end variable

When it finally reaches the new position that will make the window fulfill the criteria, we process the result of the new sliding window. Maybe we need the length to find a maximum or minimum among all the substrings that fit the criteria, maybe we need to find if the window contains a certain character and note the string that the window forms. Whichever the case, once processed, remember to exit the loop.

However, there are times when you do not need this inner while loop. This happens when the length of the window is specified. For example, finding the substring of length X with the most number of vowels. In this case, there is no need to shift the start of the window constantly in search of the more optimum position of the window since the length of the window has to be fixed.

Instead, we just need to shift once, and use an if statement to check if the window has gone beyond its size.

We also do not need the window_start position of the window since we can derive it from the window_end position and the required length of the window.

When that happens, we need to remove the effect of the element at the start of the window, and then analyse what’s left of the window. Remember, at this point, the current element has been processed and already forms part of the window. Don’t process the current element here again!

Next, line 11 will increment the end of the window as we continue sliding it towards the end of the string.

Credits to this video below which clearly explains this concept. We will write out the algorithms to solve some questions involving the different variants as mentioned above.

https://youtu.be/MK-NZ4hN7rs

So let’s try the example of fixed window length. Find the substring(s) of length 5 that has the most number of vowels. For simplicity sake, we assume the given string will have at least 5 characters and they are all lowercase.

```ruby
def sw s, k = 5
  first_string = s[0...k]
  strings = [first_string]
  max = first_string.count('aeiou')
  
  window_end = k
  while window_end < s.length
    string = s[window_end - k + 1..window_end]

    count = string.count('aeiou')
    if count > max
      max = count
      strings = [string]
    elsif count == max
      strings << string
    end

    window_end += 1
  end

  strings
end
```

The code snippets are organized following the signature of the window sliding algorithm as presented above.

Line 2 to 4, we setup the variables needed for the loops later.

In line 6, based on the question and with the prior knowledge, we skip the first qualified substring and save on some iterations in case k gets insanely big.

Line 8 is where we process the current element first. Here we form the string with the current element as the last character.

In line 11, we check the condition: does the string have a higher vowel count than our maximum.

Lines 11 to 16, we process the result. Fairly straightforward here. The key thing to note is that there is no window_start. We do not need to continue looping to find a better start of the window as the size of it is fixed. So no inner while loop.

Remember to increment window_end for the loop to propagate, and lastly return the answer in line 21.

Now let’s look at a question that requires a moving window_start point.

Let’s say you want to find the number of contiguous elements in an array of item that can sum up to reach certain value. These elements can be of any length, but they must be adjacent to each other in the sequence.

```ruby
def solution(arr, sum)
    count = 0

    window_start = 0
    window_end = 0
    while window_end < arr.length
        window = arr[window_start..window_end]

        loop do
            count += 1 if window.reduce(:+) == sum

            break if window_start == window_end

            window_start += 1
            window = arr[window_start..window_end]
        end

        window_end += 1
    end

    count
end
```

The gist lies in lines 9 to 16. A loop that continuously shift the start of the window to the right and continuously check the condition to improve the answer.

The key thing here is deciding how the loop is going to start and end, its break condition, and making sure to take care of the scenarios occurring at the edge of the window.

## Next lexicographical permutation algorithm

The key idea here is to recognize that the last permutation in lexicographical order occurs when the last character is the smallest while the first character is the largest.

In other words, when the string is lexicographically reversed. And this holds true for substrings, or suffix in particular.

Starting from the end of the string, as we move forward character by character, the moment this reversed lexicographical order gets “reversed”, it means that the index we are at marks the start of a new substring that needs to be in a sorted order.

def solution s
  index = s.length - 1
  small = nil
  while index > 0 # IMPT
    if s[index - 1] >= s[index] # IMPT
      index -= 1
    else
      small = index - 1
      break
    end
  end

  return "no answer" if small.nil?

  char = s[small]
  array = s[small...s.length].chars.sort
  offset = 1
  small_index = array.index(char)
  while array[small_index + offset] == char
    offset += 1
  end

  front = array.delete_at(small_index + offset)
  s[0...small] + array.unshift(front).join('')
end
We set index to be the index of the last character in line 2, then we loop it forward. As we are constantly comparing the character in front of index, we do not end the iteration at the start of the string as there will not be anything in front to compare.

Take note of line 5 where we also ignore the case where the value before is equal to the value at the current index. There is no lexicographical difference between same characters.

In line 8 and 9, we meet the scenario we are looking for, a “reverse” in the reversed lexicographical order of the suffix we are currently iterating through. We assign the index before the current one to the variable small and break the loop to save on the iteration count.

We do an early return in line 13, where the string is already lexicographically reversed and is already in its last lexicographical permutation.

In lines 15 to 21, we are going to find the next lexicographical permutation for the substring we are in.

The idea here is simple. As it stands, we are at the last lexicographical permutation of the suffix with the first character (let’s call this first_character being whatever the index of the string is. So the next permutation has to start with another character that is next in the lexicographical order. Once the new first character is established, the rest of the suffix just need to be sorted as we usually do and we are done.

To find this next character, sort the suffix in ascending order (which should be the lexicographical order) and identify the index of the first_character.

This character will never be the last character in the sorted suffix due to that “reversal” as we were looping from the end of the original string in lines 4 to 10. We always know that there’s at least 1 more character after it.

If the string is made up of unique characters, the next character after it will be the new first character of the new suffix to make up the next lexicographical permutation of the original string.

However, if there can be duplicates in the original string, then we have to make sure the next character has to be different. With that, we use an offset, which will be minimum of value 1 since there’s always a next value and we do not need to worry about hitting out of range.

Line 19 to 20, we start from where the first_character is in the sorted suffix, and continuously increase the offset until the we find a different character that the first_character. Once again, we are looping along a sorted suffix here, and there will always be a character larger than the first_character, so we will definitely find the next character to start the suffix.

With this new character to mark the new suffix, we just have to join it back with the prefix and we will get the next lexicographical permutation. Line 23 and 24 shows just one way to do this.

## Longest Palindromic Substring Using Manacher’s Algorithm

The common algorithm to solve for the longest common substring is the Manacher’s algorithm. There are many tutorials that explain it better than I will ever be able to, so I would not be covering the exact logic behind it. The best video explaining this concept is added below.

https://youtu.be/nbTSfrEfo6M

I would, however, be covering the gist of the code largely for my revision and refresher purpose. But before that, I will like to quickly remind myself what Manacher’s algorithm does not solve.

The Manacher’s algorithm only solves for the length of the longest substring that is a palindrome in a given string. It can also give you information of how many longest palindromic substrings are there.

The Manacher’s algorithm does not tell you what substrings they are nor the indices of where they start or where their centers are.

With that in mind, let’s hop into the algorithm.

```ruby
def manacher string
  s = "|#{string.chars.join('|')}|"
  array = Array.new(s.length) {0}
  c = r = 0

  i = 1
  while i < s.length - 1 # IMPT
    if i < r
      array[i] = [
        r - i,
        array[2 * c - i]].min # IMPT
    end

    
    loop do
      offset = array[i] + 1
      break if s[i - offset] != s[i + offset]

      array[i] += 1
    end

    if i + array[i] > r # IMPT
      c = i
      r = i + array[i]
    end

    i += 1
  end

  array.max
end
```

We start off by creating the modified string, s, that has special characters on either side of each character. This is required to ensure all substrings have odd number of characters in them so that there is only 1 true center. Note that these special characters will take part in the formation of the palindrome, but in a genius manner, they do not affect the result.

An array of equal length to the string s is also created to hold the number of characters that match on either side of the corresponding character of the string at each index.

We set c as the center of the possible palindromes that we will be investigating, and r to remember the location of the edge of the palindrome that we have processed at each iteration.

We loop through the string s in lines 7 to 28 starting from index 1 and ending 1 before the last. This is because the characters at either end of the string will not form a proper center of a palindrome and thus there is no need to process them.

Remember, in this algorithm, we are constantly finding new center of possible palindromes and ascertaining the possible lengths of these new centers. As a center, it has to have a least a character on either side. That’s why the first and last characters are disqualified.

In each iteration within lines 7 and 28, we have to check 3 cases.

The first case in lines 8 to 12, we check if we are still within the right edge of the palindrome that we calculated in the previous iteration with center c. If so we can simply copy over the value at the mirror index of the current index i, pivoted around the center c, but with a caveat that we will cover shortly.

The idea here is that within the palindrome of center c, what’s on the right is the same as the left. Hence, if there are sub-palindromes on the right side of c, they will already have been calculated on the left side. We can thus copy over their lengths that have already been recorded in the array variable. 2 * c - 1 will give the mirror index.

However, this holds true only if the length of the palindrome previously recorded at the mirror index does not exceed the right edge of the palindrome of the current iteration. This is because it is possible that the next character after the right edge of the palindrome can chain and form a longer palindrome than what was previously calculated.

If that happens, the minimum length of this potentially longer palindrome is going to be at least r - i.

Hence, we set the value at i of array to the minimum of those 2 lengths. Note that this is not the final answer, we still have to calculate and find out if there’s a longer palindrome that can be formed.

The second case in lines 15 to 20, we ascertain how long a palindrome can be formed with character at i as the center. We compare the characters at an offset on either side of the center i and continually increase until they do not match.

The array[i] sets the offset at a starting value and skips repeated calculations that characters that we have already confirmed to be palindromic with the center i.

Lastly, we check if the new palindrome with the center i extends beyond the right edge of the current palindromic center c. If it does, we set the new values of c and r since they hold the latest data and allow us to iterate further.

Line 27, remember to increase i!

The array now contains the lengths of longest palindromic substring that each character at each index can form as the center. Use the max function to find the answer.

To find the number of longest palindromic substring, find the max value in the array and simply count how many times it appeared.

## Anagrams

TODO

## Palindrome

TODO
