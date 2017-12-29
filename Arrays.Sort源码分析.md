# Arrays.Sort源码分析



## 算法概览：

1. 当数组长度小于 286时,使用[三路快速排序](http://www.geeksforgeeks.org/dual-pivot-quicksort/)
2. 当数组长度大于286时使用归并排序
3. merge sort 或者quick sort  divide 的子数组的长度小于47时，利用insert sort 将子数组进行排序。


## 疑问： 
1. **为什么small array 利用insert sort 的效率更高？为什么将阈值定为47**
    1.  Insertion sort is also good because it's useful in online situation, when you get one element at a time. 
        也就是说，归并排序和快速排序都假设待排序数组都已经存在于内存中
    2.  Insertion sort's inner loop just happens to be a good fit for modern CPUs and caches -- it's a very tight loop that accesses memory in increasing order only
    3.  An important concept in analysis of algorithms is**asymptotic**analysis. In the case of two algorithms with different asymptotic running times, such as one O(n^2) and one O(nlogn) as is the case with insertion sort and quicksort respectively,**it is not definite that one is faster than the other.**The important distinction with this sort of analysis is that for **sufficiently large N**, one algorithm will be faster than another. When analyzing an algorithm down to a term like O(nlogn), you drop constants. When realistically analyzing the running of an algorithm, those constants will be important only for situations of small n.
      So what does this mean? That means for certain small n, some algorithms are faster. This [article](http://embeddedgurus.com/stack-overflow/2009/03/sorting-in-embedded-systems) from EmbeddedGurus.net includes an interesting perspective on choosing different sorting algorithms in the case of a limited space (16k) and limited memory system. Of course, the article references only sorting a list of 20 integers, so larger orders of n is irrelevant. Shorter code and less memory consumption (as well as avoiding recursion) were ultimately more important decisions. Insertion sort has low overhead, it can be written fairly succinctly, and it has several two key benefits: it is stable, and it has a fairly fast running case when the input is nearly sorted.

#### dual-piovt quick sort

![image.png](http://upload-images.jianshu.io/upload_images/5247090-b53d8dd9aba4368c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

1.  当数组长度小于设定的阈值时，利用插入排序对数组进行排序
2.  选择两个核心元素，p1 和p2。例如利用最左侧的元素作为p1，最右侧元素作为p2
3.  p1 必须小于p2，不然的话，swap
4.  根据p1,p2将数组分割为上面的4部分，结果为上图所示，part 1 ,part 2, part 3比较明显分别，Part 4 即
   index在[K,G]
   之间的元素是待放置的元素。
5.  将K,G之间的元素放置到该放置的地方后，将P1的值和 Part 1的最后一个元素互换，P2 和 Part 3的第一个元素交换
6.  重复进行第 1-6 步

其实这就是快速排序的改进后的三路排序算法 [LeetCode上面的一个题](https://leetcode.com/problems/sort-colors/description/)

## 一些细节算法代码片段

### 检查数列是否有序

```java
       // Check if the array is nearly sorted
        for (int k = left; k < right; run[count] = k) {
            if (a[k] < a[k + 1]) { // ascending
                while (++k <= right && a[k - 1] <= a[k]);
            } else if (a[k] > a[k + 1]) { // descending
                while (++k <= right && a[k - 1] >= a[k]);
                for (int lo = run[count] - 1, hi = k; ++lo < --hi; ) {
                    int t = a[lo]; a[lo] = a[hi]; a[hi] = t;
                }
            } else { // equal
                for (int m = MAX_RUN_LENGTH; ++k <= right && a[k - 1] == a[k]; ) {
                    if (--m == 0) {
                        sort(a, left, right, true);
                        return;
                    }
                }
            }

            /*
             * The array is not highly structured,
             * use Quicksort instead of merge sort.
             */
            if (++count == MAX_RUN_COUNT) {
                sort(a, left, right, true);
                return;
            }
        }
```
上面用来检查数组是否是一个已经高度排序的算法,当未高度排序的数量大于MAX_RUN_COUNT(67)，或者元素相等的数量大于MAX_RUN_LENGTH(33)时，直接使用归并排序进行排序


