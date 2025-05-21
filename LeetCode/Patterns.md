1. Prefix Sum 

![[Pasted image 20250513174450.png]]

Prefix Sum involves **preprocessing** an array to create a new array where each element at index `i` represents the sum of the array from the start up to `i`. This allows for efficient **sum queries on subarrays**.

Use this pattern when you need to perform multiple sum queries on a subarray or need to calculate cumulative sums.

#### Sample Problem:

Given an array `nums`, answer multiple queries about the sum of elements within a specific range `[i, j]`.

**Example**:

- **Input:** `nums = [1, 2, 3, 4, 5, 6]`, `i = 1`, `j = 3`
    
- **Output:** `9`
    

#### Explanation:

1. Preprocess the array `A` to create a prefix sum array: `P = [1, 3, 6, 10, 15, 21]`.
    
2. To find the sum between indices `i` and `j`, use the formula: `P[j] - P[i-1]`


Prefix Sum involves **preprocessing** an array to create a new array where each element at index `i` represents the sum of the array from the start up to `i`. This allows for efficient **sum queries on subarrays**.

Use this pattern when you need to perform multiple sum queries on a subarray or need to calculate cumulative sums.

#### Sample Problem:

Given an array `nums`, answer multiple queries about the sum of elements within a specific range `[i, j]`.

**Example**:

- **Input:** `nums = [1, 2, 3, 4, 5, 6]`, `i = 1`, `j = 3`
    
- **Output:** `9`
    

#### Explanation:

1. Preprocess the array `A` to create a prefix sum array: `P = [1, 3, 6, 10, 15, 21]`.
    
2. To find the sum between indices `i` and `j`, use the formula: `P[j] - P[i-1]`

![[Pasted image 20250514113317.png]]


We can manipulate the same array as 

for(int i = 1; i <  arr.length-1; i++){
	arr[i] += arr[i-1]; // its like finding the cumulative frequency in the stat...
}


2.  Two pointer
3. Sliding window
