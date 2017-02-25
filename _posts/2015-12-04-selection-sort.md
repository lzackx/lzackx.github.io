---
layout: post
title:  "选择排序：算法系列（四）"
date:   2015-12-04 09:00:00
categories: Algorithm
---
**选择排序（Selection Sort）算法步骤**

1. 把整组数据集合看待成无序子集。
2. 从数据集合的首位数据标记最小（或最大，根据排序规则而定）数据的下标，然后开始往后遍历数据，当发现比标记的数据小的数据时，标记改为较小的数据的下标。
3. 把数据遍历完一遍后，把标记了下标的数据按照从数据集合首位往后替换放置。
4. 重复步骤2、3直到无序子集内无数据。

选择排序算法示例代码（Swift语言）：

	func selectionSort(inout array:[Int]){
	    
	    var i:Int,j:Int;
	    var target:Int;        //标记数据集合中目标数据的下标
	    var exchange:Int;
	    
	    for(i = 0;i < array.count;i++){
	        target = i;
	        //当遍历出比现在标记了的元素要小的数据时，替换标记
	        for(j = i+1;j < array.count;j++){
	            if(array[j] < array[target]){
	                target = j;
	            }
	        }
	        exchange = array[target];
	        array[target] = array[i];
	        array[i] = exchange;
	    }
	}

	var a:[Int] = [0,2,4,6,8,9,7,5,3,1];    //[0, 2, 4, 6, 8, 9, 7, 5, 3, 1]
	selectionSort(&a);                      //[0, 1, 2, 3, 4, 5, 6, 7, 8, 9]

选择排序算法时间复杂度：

1. 已经按照规则排序好的一组数据

	C(min) = 1 + 2 + 3 + ... + (n - 1) = (n x (n - 1)/2) ＝ O(n^2)
	
	M(min) = 0

2. 需要全排序的一组数据

	C(max) = 1 + 2 + 3 + ... + (n - 1) = (n x (n - 1)/2) ＝ O(n^2)
	
	M(max) = (n - 1) x 3 = O(n)

&emsp;&emsp;**总结**:选择排序是一种思维与插入排序相似，但前后操作顺序不同的排序算法，选择插入排序是挑了数据再找位置，而选择排序是找了位置才挑数据。但选择排序是一种不稳定的排序算法。
