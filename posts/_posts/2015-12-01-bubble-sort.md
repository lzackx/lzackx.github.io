---
layout: post
title:  "冒泡排序：算法系列（一）"
date:   2015-12-01 09:00:00
categories: Algorithm
---
**冒泡排序（Bubble Sort）算法步骤**

1. 从数组首位及其后一位数据开始进行比较。
2. 两者较大（或较小，根据排序方式）的往一边放置（例如大的放在数组id大的位置，小的放在数组id小的位置）。
3. 把数组遍历排序一次后，最大（或最小）的数据就被放在了其中一边。
4. 每一次减少一个已经排好在一边的数据，重复第三步骤，可完成冒泡排序。

冒泡排序算法示例代码（Swift语言）：

	func bubbleSort(inout array:[Int]){
	    
	    /*
	    **  因为是每次两数据进行比较，(array.count - 1)可以用“种树问题”来理解
	    */
	    for(var i = 0;i < array.count - 1;i++){
		/*
	        **  1.也因为是每次两数据进行比较，(array.count - 1)可以理解为在“末尾减一”后停止两数据对比
	        **  2.之后的(-i)是因为每遍历完一次数组，就等于把一个数据排序完毕，在下一次排列种不参与遍历
	        */
	        for(var j = 0;j < ((array.count - 1) - i);j++){
	            if(array[j] > array[j+1]){
	            	//加减法交换
	                array[j+1] += array[j];
	                array[j] = array[j+1] - array[j];
	                array[j+1] = array[j+1] - array[j];
	            }
	        }
	    }
	}

	var a:[Int] = [0,2,4,6,8,9,7,5,3,1];    //[0, 2, 4, 6, 8, 9, 7, 5, 3, 1]

	bubbleSort(&a);	                        //[0, 1, 2, 3, 4, 5, 6, 7, 8, 9]

冒泡排序算法时间复杂度：

1. 已经按照规则排序好的一组数据

	C(min) = n - 1	    //进行了（n - 1）次的比较(Compare)
	
	M(min) = 0	    //没有进行交换动作(Move)

2. 需要全排序的一组数据

	C(max) = 1 + 2 + 3 + ... + (n - 1) = (n x (n - 1)/2) ＝ O(n^2)
	
	M(max) = C(max) x 3 = O(n^2)

&emsp;&emsp;**总结**:冒泡排序是很简单的排序算法，形象地用水泡从水中往水面冒出的过程，Coder必须掌握的一种基础算法。