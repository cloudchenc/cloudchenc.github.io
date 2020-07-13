---
title: 剑指 Offer 40. 最小的k个数
tags:
    - LeetCode
    - 剑指Offer
    - 快速排序
    - 堆排序
    - 二叉搜索树
    - 计数排序
categories: 剑指Offer
---

快排：O(N)
```java
class Solution {
    public int[] getLeastNumbers(int[] arr, int k) {
        if(k == 0 || arr.length == 0){
            return new int[0];
        }
        return quickSearch(arr, 0, arr.length - 1, k - 1);
    }

    private int[] quickSearch(int[] nums, int low, int high, int k){
        int j = partition(nums, low, high);
        if(j == k){
            return Arrays.copyOf(nums, j+1);
        }
        return j > k ? quickSearch(nums, low, j - 1, k):quickSearch(nums, j+1, high, k);
    }

    private int partition(int[] nums, int low, int high){
        int pivot = nums[low];
        int i = low, j = high + 1;
        while(true){
            while(++i <= high && nums[i] < pivot);
            while(--j >= low && nums[j] > pivot);
            if(i >= j){
                break;
            }
            int temp = nums[j];
            nums[j] = nums[i];
            nums[i] = temp;
        }
        nums[low] = nums[j];
        nums[j] = pivot;
        return j;
    }
}
```

堆排：O(NlogK)
```java
class Solution {
    public int[] getLeastNumbers(int[] arr, int k) {
        if(k == 0 || arr.length == 0){
            return new int[0];
        }
        
        Queue<Integer> pq = new PriorityQueue<>((v1, v2) -> v2 - v1);
        for(int num : arr){
            if(pq.size() < k){
                pq.offer(num);
            }else if(num < pq.peek()){
                pq.poll();
                pq.offer(num);
            }
        }

        int[] result = new int[pq.size()];
        int index = 0;
        for(int tmp : pq){
            result[index++] = tmp;
        }
        return result;
    }
}
```

二叉搜索树：O(NlogK)  
BST 相对于前两种方法没那么常见，但是也很简单，和大根堆的思路差不多～
要提的是，与前两种方法相比，BST 有一个好处是求得的前K大的数字是有序的。

因为有重复的数字，所以用的是 TreeMap 而不是 TreeSet
```java
class Solution {
    public int[] getLeastNumbers(int[] arr, int k) {
        if(k == 0 || arr.length == 0){
            return new int[0];
        }
        
        TreeMap<Integer, Integer> map = new TreeMap<>();
        int count = 0;
        for(int num:arr){
            if(count<k){
                map.put(num, map.getOrDefault(num, 0)+1);
                count++;
                continue;
            }

            Map.Entry<Integer, Integer> entry = map.lastEntry();
            if(entry.getKey() > num){
                map.put(num, map.getOrDefault(num, 0) + 1);
                if(entry.getValue() == 1){
                    map.pollLastEntry();
                }else {
                    map.put(entry.getKey(), entry.getValue()-1);
                }
            }
        }

         int[] result = new int[k];
            int index = 0;
            for(Map.Entry<Integer, Integer> entry : map.entrySet()){
                int freq = entry.getValue();
                while(freq -- > 0 ){
                    result[index ++] = entry.getKey();
                }
            }
            return result;
    }
}
```

当数据范围有限时计数排序：O(N)
```java
class Solution {
    public int[] getLeastNumbers(int[] arr, int k) {
        if(k == 0 || arr.length == 0){
            return new int[0];
        }
        
        int[] counter = new int[10001];
        for(int num:arr){
            counter[num]++;
        }

         int[] result = new int[k];
            int index = 0;
            for(int i = 0; i < counter.length ; i++){
                while(counter[i]-- > 0 && index < k){
                    result[index++] = i;
                }

                if(index == k){
                    break;
                }
            }
            return result;
    }
}
```

参考资料:

https://leetcode-cn.com/problems/zui-xiao-de-kge-shu-lcof/solution/3chong-jie-fa-miao-sha-topkkuai-pai-dui-er-cha-sou/