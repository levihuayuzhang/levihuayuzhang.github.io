<!-- ---
layout: post
title:  "Classic Sorting Algorithms"
date:   2023-12-21 15:26:30 +0800
tags: 
    - algorithms
    - datastructure
categories:
    - algorithms
toc: true
--- -->
# Classic Sorting Algorithms

[LeetCode 912](https://leetcode.cn/problems/sort-an-array/)

[Complexity info](https://en.wikipedia.org/wiki/Sorting_algorithm#Comparison_of_algorithms)

### Bubble sorting

The whole process look like bubbles going up to surface.
While sorting (target to acending order), the most right element becomes the largest one.

<!-- more -->

```cpp
class Solution {
public:
    vector<int> sortArray(vector<int>& nums) {
        if (nums.size() == 0) {
            return nums;
        }

        // bubble sort
        for (int i = 0; i < nums.size(); i++) {
            // -1 because compares to next element
            // -i because the right most element is the least int after every loop
            for (int j = 0; j < nums.size() - 1 - i; j++) {
                if (nums[j] > nums[j+1]) {
                    int tmp = nums[j];
                    nums[j] = nums[j+1];
                    nums[j+1] = tmp;
                }
            }
        }

        return nums;
    }
};
```


### Select Sort

Keep finding the most less element and swap with the most left one in sub array (ascending order).


```cpp
/*select sort*/
class Solution {
public:
    vector<int> sortArray(vector<int>& nums) {
        if (nums.size() == 0) { // no element for sorting
            return nums;
        }

        for (int i = 0; i < nums.size(); i++) {
            int min = i; // assume the left is the least

            for (int j = i; j < nums.size(); j++) { // start from i not 0
                if (nums[j] < nums[min]) { // keep finding the least one
                    min = j;
                }
            }
            // swap the left most and the min one in the sub array
            int tmp = nums[i];
            nums[i] = nums[min];
            nums[min] = tmp;
        }

        return nums;
    }
};
```

### Insertion Sort

Use double pointer, one points to last element of sorted sub array, one for the comparing element [i+1].

If the element is larger, move backward. If not, insert the current value into the position.


```cpp
// insertion sort
class Solution {
public:
    vector<int> sortArray(vector<int>& nums) {
        if (nums.size() == 0) {
            return nums;
        }

        int cur; // record current value

        for (int i = 0; i < nums.size() - 1; i++) { // no need to compare last element
            // compare start from nums[i + 1]
            // i stand for the compared sub array
            int cur = nums[i + 1]; 
            int tmpIndex = i; // the right index for current element in sub array
            // move elements backward
            while (tmpIndex >= 0 && cur < nums[tmpIndex]) {
                nums[tmpIndex+1] = nums[tmpIndex];
                tmpIndex--;
            }
            // right position (tmpIndex + 1) in sub array
            nums[tmpIndex + 1] = cur;
        }

        return nums;
    }
};
```

### Quick Sort

#### classic (one pivot)

1. select a pivot (randomly or just pick as you like).
2. Sort the array, put the elemtnts that less than pivot to left and larger to right, then we have two sub array divided by pivot
3. sort recursively sort the sub array until the array size go to zero
4. do all above in the original array to save space:
    - move pivot to `right` first
    - sort element in `left` to `right - 1`
    - swap middle first element of larger array (`i+1`) with pivot (`right`)
    - then, `i+1` is the middle point the 'true' `pivot`, return it as the divide point for next recursion

```cpp
// quick sort
class Solution {
    int partition(vector<int>& nums, int left, int right) {
        int index = (right - left) / 2 + left; // middle point as pivot
        swap(nums[right], nums[index]); // move pivot to last position
        
        int pivot =  nums[right];
        int i = left - 1; // i for last element in left sub array (less than pivot)

        // j for index for comparing
        // right - 1 that left most right element (pivot) for swaping at last step
        for (int j = left; j <= right - 1; ++j) { 
            // swap all elements that smaller than pivot to front sub array
            if (nums[j] <= pivot) { 
                swap(nums[++i], nums[j]);
            }
        }
        // for first elements that lager than pivot in right sub array,
        // which should be the position of pivot
        swap(nums[i+1], nums[right]); // move pivot to middle point
        return i+1;
    }


    void quick_sort(vector<int>& nums, int left, int right) {
        if (left < right) {
            int pos = partition(nums, left, right); 
            quick_sort(nums, left, pos - 1);
            quick_sort(nums, pos + 1, right);
        }
    }

public:
    vector<int> sortArray(vector<int>& nums) {
        quick_sort(nums, 0, (int)nums.size() - 1);
        return nums;
    }
};
```

### Shell Sort

<!-- Best time complexity: $$O(nlogn)$$ -->
<!-- https://rust-lang.github.io/mdBook/format/mathjax.html?highlight=math#mathjax-support -->
Best time complexity: \\( O(nlogn) \\)

Add a gap compare to insertion sort:

- every time compare the element that has distance of gap
- reduce the gap (by dividing 2 or using other method) until the gap equal to 1 (when gap euqal to 1, it becomes the classic insertion sort).
- the introduce of gap make the insertion "faster".

```cpp
// shell sort
class Solution {
public:
    vector<int> sortArray(vector<int>& nums) {
        int cur;
        int gap = nums.size();
        while (gap > 0) { // reduce gap until gap equal to 1
            // perform insertion sort within the gap
            for (int i = gap; i < nums.size(); i++) {
                cur = nums[i];
                int pre = i - gap;
                while (pre >= 0 && cur < nums[pre]) {
                    nums[pre+gap] = nums[pre];
                    pre -= gap;
                }
                nums[pre+gap] = cur;
            }
            gap /= 2; // reduce the gap
        }
        return nums;
    }
};
```

### Merge Sort

Divide and Conquer:

1. recursively divide the original array into two parts
2. sort the sub array and merge the result into one

```cpp
// merge sort
class Solution {
    vector<int> merge(vector<int> left, vector<int> right) {
        vector<int> result;

        for (int index=0, i=0, j=0; index < left.size()+right.size(); index++) {
            if(i >= left.size()) { // left merge finished
                result.push_back(right[j++]);
            } else if(j >= right.size()) { // right merge finished
                result.push_back(left[i++]);
            } else if (left[i] < right[j]) { // put the smaller into the result
                result.push_back(left[i++]);
            } else {
                result.push_back(right[j++]);
            }
        }
        
        return result;
    }

public:
    vector<int> sortArray(vector<int>& nums) {
        if (nums.size() < 2) {
            return nums;
        }

        // keep devide the array into two sub array
        int mid = nums.size() / 2;
        vector<int> left (nums.begin(), nums.begin()+mid);
        vector<int> right (nums.begin()+mid, nums.end());
        
        return merge(sortArray(left), sortArray(right)); // merge the result
    }
};
```

### Heap Sort

Use heap to maintain the array.

Heap: complete binary tree， besides, each root node of sub tree larger or less than its child nodes.

1. make the array to a heap (if ascending, make a big top one)
2. swap the root in the heap with the last element in the array (so that the last element is at the target position)
3. adjust the array again to a heap (so that the root is always the largest or the least value)
4. keep doing step 2 and 3

The root node always record the largest or least value, so it can be swap to the end of the array to make the array ordered.

```cpp
// heap sort
/*  for element at k: 
    left node index is 2*k+1
    right node index is 2*(k+1), which is 2*k+2

    last none leaf node index is N/2-1 (N is the length of array)
*/
class Solution {
    // adjust sub trees to make the binary tree to a heap (again)
    // i for the root node of sub tree (last non leaf node)
    void maxHeapify(vector<int>& nums, int i, int len) {
        while ((i<<1)+1 <= len){ // i<<1 is i*2, which is faster
            int lson = (i<<1) +1;
            int rson = (i<<1) +2;
            int large;

            if (lson <= len && nums[lson] > nums[i]) {
                large = lson;
            } else {
                large = i;
            }

            if (rson<= len && nums[rson] > nums[large]) {
                large = rson;
            }
            // the root node of sub tree is not larger than child node, swap,
            // so that it becomes a heap
            if (large != i) {  
                swap(nums[i], nums[large]);
                i = large; // update index in the array
            } else {
                break;
            }
        } 
    }

    // initialize the heap
    void buildMaxHeap(vector<int>& nums, int len) {
        for (int i = len / 2; i >= 0; --i) {
            maxHeapify(nums, i, len); // i is the last non leaf node
        }
    }

    void heapSort(vector<int>& nums) {
        int len = nums.size() - 1;
        buildMaxHeap(nums, len); // initialize the heap first
        
        for (int i = len; i>= 1; --i) {
            // swap the largest nums[0] with the last element (becomes ascending order)
            swap(nums[i], nums[0]); 
            len--; // reduce range
            maxHeapify(nums, 0, len); // keep adjust the heap
        }
    }
public:
    vector<int> sortArray(vector<int>& nums) {
        heapSort(nums);
        return nums;
    }
};
```

### Counting Sort (Bucket Sort)

1. use an additional array to record the quantity of elements that shows up
2. use the counting array to put the elements in order

Limites:

1. only suite for the data set that in a tiny range with repeated value (if large range, space complexity could be bad)
2. only suite for counting integer (sorting integer)

### Bucket Sort

1. assume the data are evenly separate.
2. divide the range into multipule bucket (for range of 1 to 100, have buckets 1~10, 11~20...)
3. sort each buckets

Limites:

1. need to know the data range so can design the suitable size of bucket, otherwise waste the space and make bucket meaningless.
2. better combine with other sorting algorithm to divide large task to small one

### Radix Sort

1. find largest element and calculate the highest place
2. put ones digit into bucket and sort each bucket, then put back
3. put tenes place into bucket and sort each bucket, then put back...

Integer, radix for 10; binary radix for 2; 8 bits ASCII char radix for 256.

### Best practice in most situation

Use quick sort first and when data size becomes small, use non-recursion algorithm like insertion sort, etc.

Heap vs quick: refactor the heap will consume more resource in the real situation.
