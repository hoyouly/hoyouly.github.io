---
layout: post
title: 数据结构和算法之美 - 排序
category: 数据结构和算法
tags: 数据结构 算法
---
* content
{:toc}

除了学习算法的原理，代码实现之外，还得知道如何评价，分析一个算法
## 如何分析一个排序算法

* 最好情况，最坏情况，平均情况的时间复杂度
知道这三种情况的原始数据是什么

* 时间复杂度的系数，阶数，常数
对于同一阶数的时间复杂度，需要把系数，常数，低阶等考虑进去

* 比较次数或者交换(或移动)次数
基于比较的排序算法，会涉及到两个操作，一种是元素的大小，一种是数据的交换或者移动

##  排序的内存消耗。
可以通过空间复杂度来衡量，
* 原地排序，
指空间复杂度为O(1)的排序算法。 冒泡，插入和选择都是这种原地排序算法

## 排序算法的稳定性
如果待排序的序列中，存在值相同的元素，经过排序后，值相同的元素之间的先后顺序不变
，这种算法就叫稳定性算法，否则叫不稳定性算法。

## 冒泡排序
* 只会对相邻两个元素进行操作
* 比较相邻两个元素大小，
* 不满足就互换，
* 一次冒泡至少让一个元素移动到他应该在的位置。重复 n 次，就完成了 n 个数据的排序工作

* 属于原地排序，并且具有稳定性，
时间复杂度： 最好情况O(n),最坏情况 O(n^2)

```java
public void bubbleSore(int[] arr){
  if(arr==null||arr.length==0){
    return;
  }
  int length=arr.length;
  for(int i=0;i<length;i++){
    boolean flag;
    for(int j=i;j<length-i-1;j++){//从第 i 个开始，一直到length-i-1的位置
      if(arr[j]>arr[j+1]){//说明前面的数据大于后面的数据，那么就需要交换，
        int temp=arr[j];
        arr[j]=arr[j+1];
        arr[j+1]=temp;
        flag=true;//有数据交换，
      }
    }
    if(!flag){//没有数据交换，直接退出。
      break;
    }
  }
}
```

### 有序度和逆序度
* 有序度： 数组中具有有序关系的元素对的个数 ，有序关系元素对用数学表达式就是  如果i>j,存在a[i]>a[j],那么a[i] 和a[j]就属于有序关系对，反之，则成为逆序关系对，即逆序度
* 满有序度： 完全有序的数组的有序度叫做满有序度
* 逆序度+ 有序度=满有序度

冒泡排序包含两个原子操作，交换和比较，每次交换，有序度加一，不管算法如何改进，交换次数总是确定的，

## 插入排序
往一个有序的数组中插入一个元素，只需要遍历数组，找到对应位置，插入进去即可。这就是插入排序
将数据中的数组分为两个区间，已排序区间和未排序区间。
初始的时候，已排序区间里面就一个，就是数组第一个元素。
插入算法的核心：取未排序区间的一个元素，在已排序区间找到合适的位置，并将其插入。并保证已排序区间数据一直有序。重复这个过程，直到为排序区间元素为空，算法结束。  
插入排序也包含两个原子操作。比较和移动
先那要插入的数据和已排序区间的数据依次比较大小，找到合适的插入位置，
找到合适的插入位置后，我们还需要将这个位置之后的元素顺序往后移一位，这样才能腾出来位置给要插入的数据。

*  属于原地排序算法，稳定性算法。
* 时间复杂度： 最好情况是O(n), 从头到尾遍历有序数组
最坏情况是O(n^2) ，数组是倒序排列
平均情况是O(n^2) ，

```java
public void insertSort(int[] arr){
  if(arr==null||arr.length==0){
    return;
  }

  int length=arr.length;
  for(int i=1;i<length;i++){//未排序区间，从第二个元素开始比较
    int value=arr[i];
    int j=i-1;
    for(;j>=0;j--){//从末尾开始比较，这样方便移动数据
      if(arr[j]>value){//说明未排序区间的值在已排序区间内部，
        //那么就需要移动已排序区间，好腾出来一个位置存放value
        arr[j+1]=arr[j];//
      }else{
        break;// 找到未排序区间中元素的位置，
      }
    }
    //已经确定 value 要插入的位置，j+1
    arr[j+i]=value;//
  }
}

```

## 选择排序
有点类似插入排序，也分未排序区间和已排序区间。不同点在于选择排序每次都会从未排序区间找到最小元素，然后放到已排序区间的末尾。
时间复杂度：
最好情况 ，最坏情况 和 平均时间复杂度也是O(n^2)
属于原地排序算法，但是不是稳定性算法。正因为如此，相对于冒泡和插入排序，选择排序就要稍逊一些
```java
public void selectSort(int[] arr){
  if(arr==null||arr.length==0){
    return;
  }

  int length=arr.length;

  for(int i=0;i<length;i++){//未排序区间
    int min=i;//默认是 i 的位置是最小值
    for(int j=i+1;j<length;j++){//遍历后面的数据，找到比 i 更小的index
      if(arr[j]<arr[min]){//说明找到更小的 index ，那么最小值就只这个j
        min=j;
      }
    }
    int temp=arr[i];
    arr[i]=arr[min];
    arr[min]=temp;
  }
}
```
## 为啥插入排序比冒泡排序更受欢迎
虽然他们两个时间复杂度一样，都是原地排序，也都是稳定性算法，但是插入排序更受欢迎一些，原因如下：
冒泡排序交换数据比插入排序复杂的多，冒泡需要三个赋值操作，而插入只需要一个移动操作，这也就是所谓的同一阶数的时间复杂度，需要把系数，常数，低阶等考虑进去，冒泡排序的常数是 3 ，而插入排序的常数是1

```java
冒泡排序中数据的交换操作：
if (a[j] > a[j+1]) { // 交换
   int tmp = a[j];
   a[j] = a[j+1];
   a[j+1] = tmp;
   flag = true;
}

插入排序中数据的移动操作：
if (a[j] > value) {
  a[j+1] = a[j];  // 数据移动
} else {
  break;
}

```

插入排序的算法思路也有很大的空间，可以参考希尔排序。

插入，冒泡和选择排序，时间复杂度是O(n^2)
归并排序和快速排序都是用了分而治之的思想，时间复杂度是O(nlogn)
## 归并排序
思想：先把一组数据分成两个部分，然后对前后分别排序，最后再合并到一起，这样数组就有序了，
是用了分治思想，就是分而治之，将大问题化解为小的子问题，小的子问题解决了，大问题就随之解决了
分治算法一般都是用递归实现的，**分治是一种解决问题的处理思想，递归是一种编程技巧**

```java
  public void mergeSort(int [] arr){
    if(arr==null||arr.length==0){
      return;
    }
    mergeSort(arr, 0 ,n-1)
  }

  public void mergeSort(int [] arr, int start ,int end){
      if(start>=end)return;
      int mid=(end+start)/2;
      mergeSort(arr, start ,mid);
      mergeSort(arr, mid ,end);
      merge(arr, start , mid ,end);
  }

  public void merge(int []arr, int start , int mid ,int end){
    //用 p 和 q 分别指数两部分的第一个元素
      int p=start;
      int q=mid+1;
      int [] array=new int[end-start+1];
      int k=0;
      //开始比较大小，
      while(start<=mid&&mid<=end){
        //把小的哪位放入到临时数组中。
        if(arr[p]>arr[q]{
          array[k]=arr[q];
          q++;
        }else {
          array[k]=arr[p];
          p++;
        }
        k++;
      }
      //判断哪个数组里面有剩余
      int i=p;
      int j=q;
      if(q<=end){
        i=q;
        j=end;
      }
      while(i<j){
        array[k]=arr[i];
        k++;
        i++;
      }
      for(int i=0;i<end-start;i++){
        arr[start+i]=array[i];
      }
  }

```
分析：
  * 属于稳定性算法
  * 时间复杂度计算 O(nlogn)
  不仅递推的代码可以写成递推公式，递归的时间复杂度也可以写成递推公式。
  假设 n 个元素的时间复杂度是T(n),那么分解成两个子集的时间复杂度就是T(x/2)
  T(x)=T(x/2)+k  k就是 merge 方法合并的时候的所消耗的时间。我们知道， merge() 合并两个有序数组，时间复杂度是O(n),所以k=O(n)
  T(x)=T(x/2)+n
      =2*（T(x/4)+n/2）+n=2*T(x/4)+2n
      =4*(T(x/8)+n/4)+2n=4*T(x/8)+3n
      =8*(T(x/16)+n/16)+3n=8*(T(x/16))+4n
      =...
      =2^k(T(k/2^k))+kn

## 快速排序
核心：找到在无序数组中找到一个数，然后将比他小的数字放在他的左边，比他大的数字放在他的右边。然后递归的对左右两边进行继续排序
```java
public void quickSort(int[] arr) {
   quickSort(arr, 0 , arr.length - 1);
}

private void quickSort(int[] arr, int low , int high) {
   int pivor;
   if (low < high) {
       //将 low high 一分为二，算出关键字，该值的位置固定，不需要变化
       pivor = parition(arr, low , high);
       quickSort(arr, low , pivor - 1);
       quickSort(arr, pivor + 1, high);
   }
}

//选择一个关键字，把他放到一个位置，使其左边 的值都小于这个值，右边的值都大于这个值
private int parition(int[] arr, int low , int high) {
   int privorKey;
   privorKey = arr[low];
   ////顺序很重要，要先从右边找
   while (low < high) {
       while (low < high && arr[high] >= privorKey) {////从后往前找到比 key 小的放到前面去
           high--;
       }
       swap(arr, low , high);
       while (low < high && arr[low] <= privorKey) {//从前往后找到比 key 大的 放到后面去
           low++;
       }
       swap(arr, low , high);

   }//遍历所有记录 low 的位置即为 key 所在位置, 且固定,不用再改变
   return low;
}

private void swap(int[] arr, int i , int j) {
   int temp = arr[i];
   arr[i] = arr[j];
   arr[j] = temp;
}
```










## 思考题
