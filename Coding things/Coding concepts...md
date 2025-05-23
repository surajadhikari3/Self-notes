
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