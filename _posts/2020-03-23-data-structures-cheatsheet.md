---
title:  "Data Structures Cheatsheet"
date:   2020-03-23 09:00:00
permalink: data-structures-cheatsheet
---

This is a concise glossary of the concepts, features and applications of various data structures.

## Motivation

Due to the coronavirus outbreaks, the major lockdowns in Europe that ensued, and the stay home quarantine I have to undergo upon return to my country, I am ceasing my digital nomad life, which I have recorded in my [Instagram account](https://www.instagram.com/victor.leong.17/). So here I am, refreshing my memory on data structures as I prepare to welcome a new phase of my life.

I have a problem finding a good concise cheatsheet that can properly remind me of the concepts of all the data structures, their key features and their runtime performance for various operations. More importantly, when to use them and, as I am a rubyist, how are they applied in ruby.

Each section will talk about 1 data structure. It will consist of the main concept behind how they are constructed, some key features that are unique to them, when it is the best use case for them, and if there is something similar in ruby. These concept follows the [HackerRank’s youtube channel’s playlist on Data Structures](https://www.youtube.com/playlist?list=PLI1t_8YX-Apv-UiRlnZwqqrRT8D1RhriX).

## ArrayList

An arraylist is a dynamic array that will expand its capacity when it reached its maximum. An array requires pre allocated memory to be created. That means we need to establish the size of each element in the array and their total count.

Typically, when the arraylist reaches capacity, its size will be doubled by some complicated built-in algorithm in one of the library files of the language. It also has methods that can be called manually to [ensureCapacity](https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/util/ArrayList.html#ensureCapacity(int)) of the array.

Array and arrayList are used interchangeably in this article.

### Key Features

- Expands capacity when required

### Runtime

- Access: O(1) with use of known index of element in array
- Search: O(n)

### Insert

- prepend: O(n) due to need to shift all elements
- append: O(1)
- Deletion: O(n) due to need for search to destroy
- 
### Applications

List of items of any kind of order

### Ruby Alternative

In ruby, everything is an object. That includes arrays. Arrays in ruby are made dynamic to behave like ArrayList, like in most other dynamic languages. The array object has some operation to ensure capacity for the array.

It is also heterogenous, which allows for different data types exists together as elements of the same array (since all of them are objects anyway).

## Binary (Search) Tree

Trees are most of the time referring to binary trees. Each node in binary tree can have a maximum of 2 nodes. This “tree” is kind of like a linked list of objects. It is not an array.

And for binary search tree (BST), it has to have an increasing order in relation to a node from its left to right nodes. Based on this rule, the binary search can be carried out by propagating through the nodes by asking the deterministic question: Is the left node more or less than the right node. With a known sort order, each iteration can, probably, halve the total nodes to search. This results in a faster search time.

This is only  “probably” achievable if the BST is balanced. If the tree is lopsided on the right side for example, each iteration does not exactly halve the number of nodes to search. The worst case scenario would be to comb through all the nodes if they are all existing on the right node of one another.

There are many self balancing trees, one of them is the [AVL tree](https://en.wikipedia.org/wiki/AVL_tree) named after its inventors. It involves changing the root node when it becomes unbalanced to ensure that [“the heights of the two child subtrees of any node differ by at most one“](https://en.wikipedia.org/wiki/AVL_tree).

Duplicates are allowed in some BST, meaning the there can be 2 nodes with the same value. It should always be obey the rule that the left node is <= to the current node. Duplicates introduce complexity in the search algorithms to determine the correct node to pick.

### Key Features

A node and its left and right nodes has to be sorted in a specific order that can be classified as ascending or descending
Needs to be balanced to be useful

Traversal is always from left node then right node, with the current node hoping between and around the left and right to define the 3 different methods of traversal, ie.

- inorder
- preorder
- postorder

### Runtime

- Balanced
  - Access: O(log n)
  - Search: O(log n)
  - Insert: O(log n)
  - Deletion: O(log n)
- Imbalanced (worst case scenario)
  - Access: O(n)
  - Search: O(n)
  - Insert: O(n)
  - Deletion: O(n)

### Applications

- Database like [CouchDB](http://guide.couchdb.org/draft/btree.html)
- [Huffman Coding Algorithm](https://en.wikipedia.org/wiki/Huffman_coding) for file compression
- Generally large data with sortable characteristic, and its size should be large enough to justify the use of BST over arrays

### Ruby Alternative

There is no native implementation of BST in ruby. However, there are gems out there that implement it. [RubyTree by evolve75](https://github.com/evolve75/RubyTree) is my favorite as it allows for content payload to be added to each node.

BST is quite an old and establish concept. Hence, these gems might appear old and unmaintained.

## Min/Max Heap

This tree always populated from the left to right across each level. It is considered minimum or maximum depending on whether the smallest or largest value is at the root node respectively.

After insertion, the new node is “bubbled” up to the correct node by a series of swapping with its parent node until it reaches the root node, if it reaches the root node.

If root node is deleted, the last node replaces it and “bubbled” down to the correct position.

Because of the way it is data is populated, there will be no gaps in between nodes, hence this tree can be stored as an array (no need for linked list)! One can simply use the index of the node in the array to access itself, and some formula to get the index of its neighbouring nodes and access them as well:

- parent: (index – 1) / 2 (rounded down)
- left: 2 * index + 1
- right: 2 * index + 2

### Key Features

- Essentially an array
- Root node is always the minimum or maximum, the last node is always the opposite
- Nodes in between does not necessarily obey  the order.
- Root node is usually the one being removed in application, replaced by the last node, and bubbled to the correct position accordingly
- Min heap always look to find the smallest value among its children to swap down, opposite for max heap

### Runtime

- Access: O(1) with use of known index of element in array
- Search: O(n)

### Insert

- append/prepend: O(1)
- ordered insert: O(log n)
- Deletion: O(log n)

### Applications

- Priority queues (eg. for elderly and disabled then healthy adults using weighted representation)
- Hospital queues for coronavirus victims based on age and, therefore, savableness
- Schedulers (eg task with higher priority will have higher weightage and will be bubbled to the correct position when added to the queue)
- Continuous median problem

### Ruby Alternative

There is no native heap implementation in Ruby. Gems are available.

## Hash Table

Interestingly, a hash table consist of a hashing function and an array of linked lists. Together, they form a key value datastore.

The key to map to the value to store undergoes a hashing function to get an integer. This integer will represent the index in the array in which to store the data, that is the value corresponding to the key in the hash table. It will be added to the linked list behind the index of that array.

The data is saved as a linked list instead of an element in the array due to the probability of collisions from the hashing function. This allows multiple values to be stored in the same index of the array, but only if their key is different. Otherwise, they will overwrite the old data, as hash tables do no allow duplicate keys.

It is crucial for the hashing function to have a good key distribution. This is to prevent any of the linked list from being overwhelmingly long, resulting in long search time hopping through the linked list. [Murmur hash](https://en.wikipedia.org/wiki/MurmurHash) is a good hashing function for this purpose.

### Key Features

- Hash function maps keys to index of array
- Array is made up of linked list to store data while avoiding collisions from the hashing
- Hash function with good distribution crucial to performance
- No order

### Runtime

- Access: O(1)
- Search: O(1)
- Insert: O(1)
- Deletion: O(1)

### Applications

- Anything that does not require order

### Ruby Alternative

[Murmur hash seems to be used in the native ruby hash](https://launchschool.com/blog/how-the-hash-works-in-ruby). Note that it is easily reversible. Hence while it can be used for maintaining good key distribution, it is not ideal for cryptographic purposes.

## Linked Lists

Each node will point to the next node. The last node will point to null. Accessing elements can be slow as the pointer need to jump through nodes, unlike array which can access instantly via the index. The advantage of linked list over array is that you do not need to allocate the required memory at the start. You will only use the memory that you need without wastage. It is very space efficient.

Another advantage is its speed during prepending elements or inserting them in the middle. Unlike the array where every element thereafter has to be shifted, it can be done in constant time in a linked list.

A variation, the doubly linked list, gives bearing to adjacent node on both ends. It allows traversal in both direction as its biggest advantage. The maintenance needed to maintain that the 2nd neighbouring node in all operations may be costly.

Last variation is the circular linked list. That said, there's a classic linked list question on how to detect if a linked list has a cycle (not necessarily circular). The solution is to use a fast pointer and a slow pointer to loop through the link list until they point to the same node in a linked list with a cycle, or null for a non circular linked list. This is simple cycle detection algorithm known as [Floyd’s tortoise and hare](https://en.wikipedia.org/wiki/Cycle_detection), and is entertainingly portrait in the video below.

Side note for me: the distance of the loop, not coincidentally but mathematically, equals to the start of the linked list to the location where the hare and tortoise meet. Again, not coincidentally but mathematically, the distance from the start of the linked list to the start of the loop equals to the distance from location where the hare and tortoise met to the start of the loop (continuing in the direction that the tortoise was originally moving in).

https://youtu.be/7ArHz8jPglw

### Key Features

- Head node may be null
- Last node will point to null
- Doubly linked list is another variation, where each node points to its previous and next node

### Runtime

- Access: O(n)
- Search: O(n)
- Insert
  - append/prepend: O(1)
  - ordered insert: O(n)
- Deletion: O(n)

### Applications

Anything that requires order and needs to save on memory

### Ruby Alternative

There is not native linked list in ruby. However, there are gems and [this by spectator](https://github.com/spectator/linked-list) is still pretty active.

## Queue

Queue is a collection of data that obeys the First In First Out (FIFO) principle.

Theoretically, as traversal is not suppose to happen in a queue, I believe that it is best implemented with linked list rather than an array. There is no resizing overheads, and no need to shift all the elements every time an element is taken out of the front of the queue.

Addition to the queue might mean having to hop through the whole link list to add the element at the back. However, I would solve this by using a circular linked list to have a grip on the first and last element, which is actually all the queue would care about. Of course, things will be different if it is a not-so-simple queue like a least recently used (LRU) implementation.

However, there are certain advantages we should consider implementing with arrays. Arrays can be cached more easily as they are consist of memory units adjacent to one another. On the other hand, a linked list consist of memory units that exist sparsely in the memory pool which hurts its caching capabilities. The reason is a TODO for me when I go beyond data structures during this revision weeks.

Nonetheless, cache engines like [Redis](https://redis.io/) has their own implementation of a [linked list (Redis List)](https://redis.io/topics/data-types/#lists) in their cache database. I do not know if this is the same caching mechanism that is affected by the sparse memory locations of a linked list, but it is probably good to know. 

### Key Features

- FIFO

### Runtime

- Insert (prepend): O(1)
- Deletion (shift): O(1)

### Applications

- Restaurant queues

### Ruby Alternative

Arrays are usually used as queues in ruby. It, however, does have a native [Queue](https://ruby-doc.org/core-2.7.0/Queue.html) class, which is meant for multi threaded operations. On top of that, it has a [SizedQueue](https://ruby-doc.org/core-2.7.0/SizedQueue.html) class to ensure the size is within capacity.

## Stacks

Stack are like the brother of queues. The only difference is they obey the Last In First Out (LIFO) principle.

Again as traversals are not supposed to happen, we can use linked list for the same advantages and considerations as explained in the “Queue” section above. And instead of appending to the end of the linked list, we will prepend to the linked list instead, where the head of the linked list represents the top of the stack. It will be constantly changing and where all the action will take place.

This ensures there is no overhead from resizing from using the array when the data gets too big, but it will need to take the need and performance in caching into consideration.

Arrays will be more suited to implement a stack than a queue. This is because all the push and pop will take place at the end of the array, unlike the queue which need to remove element from the front of the array and cause shifting of all elements forward.

Two stacks can be used to implement a queue with minimal performance overhead as well, as shown in the video below.

https://youtu.be/7ArHz8jPglw

### Key Features

- LIFO

### Runtime

- Insert (push): O(1)
- Deletion (pop): O(1)

### Applications

- Matching balanced parenthesis problems
- Anagram / palindrome problems
- Backtracking in maze
- Reversals

### Ruby Alternative

No native stack in Ruby. A simple array will suffice. Linked list gems are available too.

## Graph

A graph is a superset of linked list. Unlike a linked list where it needs to be associated to a next element, and its previous element for a doubly linked list, a graph can have links to multiple nodes, not just to the adjacent ones.

The link between each node of a graph contains data to give more meaning to the relationship between nodes. In graph terminology, this link is called an “edge” (as a matter of fact, “nodes” are termed “vertices“). An edge can be directed or undirected. Think being friends (undirected) versus being a follower (directed) between users in a social network.

2 common ways to search a graph is the Depth First Search (DFS) and Breadth First Search (BFS). DFS has a weakness where it will search the full depth from one edge before moving on to the next. This translates to inefficiency if the vertex that we are searching for is on the other edge. Hence BFS is preferred.

Typically in BFS, a queue is used to store the next vertices to search.

There can be cycles of vertices having an edge to one another, hence during the search, it is imperative to check if a vertex has been visited or not to prevent going round in circles during the search. Unless we are talking about a [Directed Acyclic Graph (DAG)](https://en.wikipedia.org/wiki/Directed_acyclic_graph) where there are no directed cycles.

### Key Features
- Nodes are termed vertices
- Edges contain data to describe the relation between vertices
- BFS preferred
- Flag to check if visited the vertex before in a search algorithm to prevent looping inside a cyclic relationship among vertices.

### Runtime

- Time complexity of a graph [depends on how the edges and vertices are stored](https://en.wikipedia.org/wiki/Graph_(abstract_data_type)). The optimal choice of storage depends on any prior knowledge of how the graph might look like.

### Applications
- Social network
- [Travelling salesman problem](https://en.wikipedia.org/wiki/Travelling_salesman_problem)
- Recommendation engine

### Ruby Alternative

There is no native graph data type in ruby. [Gems](https://github.com/monora/rgl) are available.

## Set

The data structure of a set is the same as that of a hash table. The difference is that set is not really concerned with the mapped value of a key. It just tracks whether the key is present.

This implies that there can be no duplicates in the keys just like a hash table. And unlike a hash table where a key can be mapped to a null value, the key will be removed if nullifed for the case of a set.

### Key Features

- No order
- No duplicates

### Runtime
- Access: O(1)
- Search: O(1)
- Insert: O(1)
- Deletion: O(1)

### Applications

- Attendance

### Ruby Alternative

Ruby has a native data type for a [set](https://ruby-doc.org/stdlib-2.7.0/libdoc/set/rdoc/Set.html).
