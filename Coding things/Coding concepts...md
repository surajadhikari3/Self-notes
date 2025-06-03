
Array Rotation Concepts

While rotating --> Most of the time the array is rotated to the right by k times..

While rotating think it is like clock after certain rotation it started to repeat like 12 hrs.

to get the rotating number of rotation % size --> For clock 15 % 12 --> 3 (means if you rotate it with 15 starting from 12 it will point to 3) 

Same analogy can be applied to the array since after certain steps it start repeating the things

number of rotation % array.length --> Which wil prevent from the unnecessary rotation and keep it preventing from indexOutOfBound error......

```
public void rotate(int[] nums, int k) {

k %= nums.length; //this is the rotation analogy and prevent from outOfBoundIndex exception..

rightShift(nums, 0, nums.length -1);

rightShift(nums, 0, k-1);

rightShift(nums,k, nums.length-1);

}

//This is robust way to rotate the array in java... instead of start loopin 
//from the arr.length-1 ;

public void rightShift(int[] nums, int start, int end){
	while(start < end){

	int temp = nums[start];// to hold the start data 

	nums[start] = nums[end];

	nums[end] = temp;

	start++;
	
	end--;
	}
}
```

Product of array except itself 

![[Pasted image 20250523110006.png]]

Here since we cannot do the division so we calculate the prefix multiplication and postfix multiplication --> Rember there is pattern called prefix sum similar manner then we multiply prefix and sufix value for getting the multiplication...........

Stack Based problems

**Reverse Polish Notation (RPN)**, also known as **postfix notation**, is a mathematical notation in which **operators follow their operands**, instead of appearing between them (as in infix notation). It eliminates the need for parentheses to define the order of operations.

![[Pasted image 20250524191545.png]]

# Monotonic stack Problem

pattern or algo used to solve problems that require tracking the next greater, previous smaller or similar relative orderings effictively in O(n) time.


## 🔍 Concept of Monotonic Stack

A **monotonic stack** is a stack that maintains its elements in either:

- **Monotonic Increasing Order** (bottom to top)
    
- **Monotonic Decreasing Order** --> For higher tempr we use this approach if the current element is greater than the top of element we pop it untill the next element is less than it and we push the current element on it.. Store the index in the stack 
    

Depending on the problem, the stack stores either:

- Values directly, or
    
- Indexes of the values

## 🧩 Real Use Cases

| Problem Name                   | Stack Type            | Order   |
| ------------------------------ | --------------------- | ------- |
| Daily Temperatures             | Monotonic Decreasing  | Temps   |
| Next Greater Element I/II      | Monotonic Decreasing  | Numbers |
| Largest Rectangle in Histogram | Monotonic Increasing  | Heights |
| Trapping Rain Water            | Two-pointers or Stack | Heights |
| Stock Span                     | Monotonic Decreasing  | Prices  |

Car Fleet problems..

Concept is that we calculate the time and if the time is same or the behind car time is same it will meet the front car and form the same fleet.

So we sort the position in descending order.Since the position is attached with the speed the speed also has to be sorted preserving the order with the position so we use 2d array.

`int[][] cars = new int[position.length][2];`  
  
```
Stack<Double> stack = new Stack<>();  
for (int i = 0; i < position.length; i++) {  
    cars[i][0] = position[i];  
    cars[i][1] = speed[i];  
}
```


Now we sort the 2d array in descending order

Arrays.sort(cars, (a, b) -> b[0] - a[0]);

Use the stack to push the value which is greater only that means it will form the fleet and cannot meet and form the same fleet.

Last we check the size of the stack.....


Sorting the Array:

We can sort the primitive array with the Arrays.sort()

But for sorting in the reverse order we have to convert the primitive type to its wrapper class and pass the comparator in the Arrays.sort()

Arrays.sort(arr, (a, b) -> b-a); --> This only works with the Wrapper class 

or Equivalent 

Arrays.sort(car, Collections.reverseOrder());

But for the 2D array we can sort directly from primitive type. This is handy while mainitaing the position on sorting like #Car-Flee-problem in the leetcode

For 2D array sorting...

`Arrays.sort(cars, (a, b) -> b[0] - a[0]);`


# Linked List

Linear data structure which have nodes (data and pointer that pointes to the next element)

linked list can be used as stack, queue and deque operation.

We use the linkedList when for frequent insertion/Deletion at any positon as th insertion/Deletion in arraylist other than the lastIndex needs the rotation of the array in the right side.


## 🧠 When to Use `LinkedList` (And Why?)

|Use Case|Reason|
|---|---|
|✅ Frequent insertions/deletions at the **beginning or middle**|`O(1)` insertion/deletion (if you have a reference)|
|✅ You don’t need random access (no frequent get(i))|`get(i)` is **O(n)**|
|❌ Avoid when frequent random access is needed|Because unlike ArrayList, it has no indexing|
|✅ Ideal for **queue**, **stack**, **deque** operations|Doubly linked nature allows fast insert/delete on both ends|

Usage ?

- Music Playlist --> For pointing to the next music (as pointer can point to next music and can be changed with O(1))

- GPS navigation

- Implement the Stack/Queue..
  
  
  Reverse Linked List
  
  
  ![[Pasted image 20250527112921.png]]
  
  `
`Code Snippets for the linkedList reversal......

ListNode current = head;  
ListNode previous = null;  
while (current != null) {  
    ListNode tmp = current.next;  
    current.next = previous;  
    previous = current;  
    current = tmp;  
}  
return previous; // the last node is the previous so we return it  
`
  
  
### Mini Analogy

Think of it like walking a hallway and turning around (changing the next to the previous) — the last door you walked through (before turning around) is now the **first door** when you reverse direction. That's why we return `previous` — it's the last node you processed, but the **first in the new direction**.

![[Pasted image 20250528100840.png]]

Basic Traversal of the linked list 

we point the current node to the next that how we move to next...


![[Pasted image 20250528105329.png]]


# Fast and slow pointer..

Use to find the linked list is cyclic or not.

we initialize the slow pointer and jump one step and the fast pointer is jumping the two step.

If the fast pointer is able to meet the slow pointer then it means it is cyclic...


# kth largest element

We can use the Min Heap which is the priority queue in the java;

Concept --> Priority queue remove the smallest element first no matter of the order // while doing poll(), take()..

Stack
Trees
LinkedList
Heap..



Java’s `PriorityQueue` is **by default a Min Heap**, but you can make a Max Heap by providing a comparator like below...

![[Pasted image 20250530111843.png]]

# BackTracking (Complex have to figure out..)


Finding the subset using the backtracking

![[Pasted image 20250530100051.png]]

1 D programming

Use the top down approach as it reaches the base condition and return the value to the method 

Use the memo--( can use map) to store the already call data preventing the deep recursion 

Another approach is to use the bottom up approach without using the memomization....


House Robber 

We have to skip the adjacent house so we have to find the max form the loot until the n-2 + current[n] and  loot untll the n-1....

We have to think the loot as a sub problem which is the concept of the dp....


 ![[Pasted image 20250602141404.png]]
 
 Now we are filling it but we have the 2 base condition and continue to fill it up...........
 
 which is 
 
 dp[0] = nums[0];
 dp[1] = Math.max(nums[0], nums[1]);
 
 ![[Pasted image 20250602141922.png]]