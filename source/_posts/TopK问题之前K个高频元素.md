---
title: TopK问题之前K个高频元素
date: 2020-04-30 15:51:58
categories: 数据结构与算法
tags: 数据结构与算法
---

给定一个非空的整数数组，返回其中出现频率前 K高的元素。

**示例 1:**

```java
输入: nums = [1,1,1,2,2,3], k = 2
输出: [1,2]
```

提示：

你可以假设给定的 k 总是合理的，且 1 ≤ k ≤ 数组中不相同的元素的个数。
你的算法的时间复杂度必须优于 O(n log n) , n 是数组的大小。
题目数据保证答案唯一，换句话说，数组中前 k 个高频元素的集合是唯一的。
你可以按任意顺序返回答案。

**解题思路：**

方法一：采用暴露排序法， **使用排序算法对元素按照频率由高到低进行排序**，然后再取前 k个元素，时间复杂度已超过nlogn。

方法二：hashMap和最小堆

- 首选用hashMap存每个数出现的次数，key为当前数，value为该数在数组出现的次数。
- 建立一个最小堆，该堆排序是按照value进行升序排序的，我们可以利用java类PriorityQueue，该类默认升序排列，该类介绍可以参看https://www.cnblogs.com/dxflqm/p/12067265.html，
- 如果建堆的时候，堆大小超过了K，我们直接poll()弹出，剩下的就是大小为K的最小堆，即为我们所求。

![alt](/Ouyang/images/acm/1.png)

代码如下：

```java
 class Solution
	{
		public List<Integer> topKFrequent(int[] nums, int k)
		{
			// 用hashMap存
			HashMap<Integer, Integer> count = new HashMap();
			for (int n : nums)
			{
				count.put(n, count.getOrDefault(n, 0) + 1);
			}

			// 初始化最小堆，按value值升序排列。
			PriorityQueue<Integer> heap =
					new PriorityQueue<Integer>((n1, n2) -> count.get(n1) - count.get(n2));

			// 遍历map，构建key，大小为K的最小key堆
			for (int n : count.keySet())
			{
				heap.add(n);
				if (heap.size() > k)
				{
					heap.poll();
				}
			}

			// 构建我们的结果
			List<Integer> top_k = new LinkedList();
			while (!heap.isEmpty())
			{
				top_k.add(0, heap.poll());
				return top_k;
			}
		}
	}
```

这个步骤需要 O(N) 时间其中 N 是列表中元素个数。

第二步建立堆，堆中添加一个元素的复杂度是 O(log(k))，要进行 N 次复杂度是 O(N)。

最后一步是输出结果，复杂度为O(klog(k)

