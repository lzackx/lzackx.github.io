---
layout: post
title:  "快速排序：算法系列（二）"
date:   2015-12-02 09:00:00
categories: algorithm
---
**快速排序（Quick Sort）算法步骤**

1. 把集合数据以集合中的某一数据为标准进行对比排序。
2. 遍历集合数据，把比标准小（或大）的数据放到标准的一边，把比标准大（或小）的数据放到标准的另一边。
3. 重复遍历数据集合，直到遍历游标重合。

快速排序算法示例代码（Swift语言,根据所用语言的特性是可以写出更简洁的代码，但为了体现算法的通用性，写成了普遍语言的样式）：

	/*
	**  array:  被排序的数据集合，想要算法兼容多种类型数据集合，可以尝试改为泛型
	**  front:  被排序数据集合的数据开始位置，使用此参数是为了方便递归调用时定位
	**  back:   被排序数据集合的数据结束位置，使用此参数时为了方便递归调用时定位
	**  Note:   front和back形参在函数内部算法具有更方便的定位方法时，可省略
	*/
	func quickSort(inout array:[Int],front:Int,back:Int){
	    
	    //在front的位置不断递增和back的位置不断递减，直到相等或相互越界后停止遍历
	    if(front < back){
	        
	        //front、back作为每一次遍历数据集合的不可变界线，因为需要让遍历，所以需要从两边界线逐渐往中间靠的变量
	        var position1:Int = front;
	        var position2:Int = back;
	        
	        //每一次的遍历都需要先定下一个比较标准，此标准可在数据集合中随便选，通常会选所排数据集合开始位置的元素
	        let standard:Int = array[front];
	        
	        //此处判断条件意思是只要不互相越界，就不断遍历集合数据
	        while(position1 < position2){
	            
	            //此处也有上面的判断条件，但重点是提供跳出循环的条件，否则一个当前排序已完成的数据集合跳不出来
	            while((position1 < position2) && (array[position2] >= standard)){
	                position2--;
	            }
	            //在position2已经递减到遇见(array[position2]<standard)情况时，把此数据覆盖了position1位置的数据
	            array[position1] = array[position2];
	            
	            /*  
	            **  在这里的循环判断语句运行前，必须明白前面出现了的array[position1]因为必定
	            **  符合(array[position2]<standard)这个条件，所以必定让position++
	            */
	            while((position1 < position2) && (array[position1] <= standard)){
	                position1++;
	            }
	            //在position1已经递增到遇见(array[position1]>standard)时，把此数据覆盖position2位置的数据
	            array[position2] = array[position1];
	        }
	        
	        //可以发现上面while循环结束后将缺少了一开始被覆盖了的开始位置元素standard，最后把其从新放回position1位置即可
	        array[position1] = standard;
	        
	        //将以standard为标准分为两部分的数据集合以递归的方式再次进行遍历，直至遇到不符合(front < back)条件位置
	        quickSort(&array, front: front, back: (position1 - 1));
	        quickSort(&array, front: (position1 + 1), back: back);
	    }
	}

	var a:[Int] = [0,2,4,6,8,9,7,5,3,1];         //[0, 2, 4, 6, 8, 9, 7, 5, 3, 1]
	quickSort(&a, front: 0, back: (a.count - 1));//[0, 1, 2, 3, 4, 5, 6, 7, 8, 9]

快速排序算法时间复杂度（推导公式过于繁杂，有空再补）：

1. 平均情况下，一层分两部分（log(n)),分n层，即O(nlog(n))。

2. 最差情况，跟冒泡一样，O(n^2)。

&emsp;&emsp;**总结**:快速排序是一种排序速度极快的算法，此处所写排序算法步骤比较简略，而示例代码注释较多，是因为算法步骤在不同的编程语言中，因为不同的语言特性可能将出现看上去完全不相同的写法能出现相同的效果，而此处所写是本质所在。而通用的算法则如示例代码所示，有些地方是需要好好斟酌感悟的，尤其替换过程。同样的，快速排序是种非常实用的算法，作为Coder必须掌握。
