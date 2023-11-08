# 26-算法系列-选择、插入排序以及STL中sort的实现

## 26.0 目录

1. 选择排序
2. 插入排序
3. STL中sort的实现
4. 资料
5. 收获

这一篇我们一起来学习实践下选择排序和插入排序，然后再一起分析下CPP的STL中排序算法的实现，结束排序算法的阶段。

## 26.1 选择排序

1. 假设一个下标对应的数组内容值为最小值（一般使用未确定的第一个），然后依次用这个值和后面的所有值进行对比大小，如果后面的值小于该值，先记录最小值的位置以及值，在不断后后续值进行比较，一次循环遍历后，根据最小值和初始最小值相比十分有变化，如果有则进行交换。
2. 下标加1，重复第一步

实现比较简单，我们就不画图，分析了，代码中加了详细的注释



```cpp
#include <iostream>
#include<array>
#include<algorithm>

using namespace std;


void printSortArray(int myarray[],int size){

        for(int k=0; k<size; k++)
        {
            cout<<myarray[k]<<" ";
        }
        cout<<endl;
}


void swap(int *a,int i,int j)
{
    int tmp = a[i];
    a[i] = a[j];
    a[j]=tmp;
}

void selectSort(int *a,int length)
{
    for(int i = 0;i<length;i++)
    {
        //先假设第一个元素为最小值，通过外部遍历记录
        int min = a[i];
        int minPos = i;

        for(int j =i+1;j<length;j++)
        {
            //进行一轮的和上述假设的最小值进行大小对比。如果出现后续的值比假定的最小值还小，则把该值赋值给最小值，
            if(min>a[j])
            {
                min =a[j];
                minPos = j;
            }
 
        }
        //一轮对比完成之后，检查最小值的下标是否发生变化，如果是则进行交换
        if(i !=minPos)
        {
            swap(a,i,minPos);
        }
        //一轮过后，最右侧的坑位被当轮的最小值只用，然后再循环确定后续的坑位的值
    }
}

int main(void) {
    //用一位数组，表示一个完全二叉树，可以从任意节点拿到它的父节点 和它的左右子节点

   int myarray[] ={6,7,1,2,10,5,8,3,4};

   int size = sizeof(myarray)/sizeof(myarray[0]) ;

   cout<<"size="<<size<<endl;

   selectSort(myarray,size);

   printSortArray(myarray,size);

    return 0;
}
```

看到选择排序，很容易想到我们前面学习实践的冒泡排序，他们之间有什么区别呐？

冒泡排序事两两相邻对比，每次对比都可能触发交互，冒泡排序是通过数找位置。
 选择排序则是先假设一个为最小值，然后用这个值和后面所有的内容一一进行比较大小，每轮进行一次交换。选择排序是先确定位置，在找值。

他们的优点都是比较简单，但是缺点也都很明显，时间复杂度是O(n^2)。选择排序还会破环原来顺序的稳定性（即 有相同值时，通过选择排序相同值的前后顺序会被破坏）。

## 26.2 插入排序

插入排序就像我们打打牌时，手里的牌是已经排序好的，起一张新的牌插入到已有的有序牌中的适当位置，为了给要插入的元素腾出空间，需要讲其余大于要插入值的元素，在插入前都向右移动一个位置。
 插入适合的应用场景：对非随机的（即有序的）队列进行插入，效率非常高。

下面我们通过画图一步步的开下插入排序的过程



![img](https:////upload-images.jianshu.io/upload_images/1791669-34f72a4484daa2ce.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

实现如下



```cpp
#include <iostream>
#include<array>
#include<algorithm>

using namespace std;


void printSortArray(int myarray[],int size){

        for(int k=0; k<size; k++)
        {
            cout<<myarray[k]<<" ";
        }
        cout<<endl;
}


void swap(int *a,int i,int j)
{
    int tmp = a[i];
    a[i] = a[j];
    a[j]=tmp;
}

void insertSort(int *a,int length)
{
    //数组下标从1开始和前面的值进行对比大小
    int i,j,tmp;
    for(i=1;i<length;i++){

        int tmp= a[i];

        for (j = i-1; j>=0; j--)
        {
            //如果后面的小于前面的有序列表中的某位置的值，则把当前位的值向后移动一位，依次循环
            if(tmp< a[j]){
                a[j+1] = a[j];

            } else {
                break;
            }
        }
       //跳出循环后，把tmp在赋值给要插入的地方
       a[j+1] = tmp;  
    }

}

int main(void) {
    //用一位数组，表示一个完全二叉树，可以从任意节点拿到它的父节点 和它的左右子节点

   int myarray[] ={6,7,1,2,10,5,8,3,4};

   int size = sizeof(myarray)/sizeof(myarray[0]) ;

   cout<<"size="<<size<<endl;

   insertSort(myarray,size);

   printSortArray(myarray,size);

    return 0;
}
```

## 26.3 STL中排序算法的实现

通过这四篇关于排序算法的的学习，我们理解了基础的选择排序、插入排序、冒泡排序、快速排序以及堆排序的原理和实现。下面我们来看下CPP中STL的排序算法的具体实现
 [源码地址]：[https://gcc.gnu.org/onlinedocs/libstdc++/libstdc++-html-USERS-4.4/a01347.html](https://links.jianshu.com/go?to=https%3A%2F%2Fgcc.gnu.org%2Fonlinedocs%2Flibstdc%2B%2B%2Flibstdc%2B%2B-html-USERS-4.4%2Fa01347.html)



```cpp
template<typename _RandomAccessIterator>
05206     inline void
05207     sort(_RandomAccessIterator __first, _RandomAccessIterator __last)
05208     {
05209       typedef typename iterator_traits<_RandomAccessIterator>::value_type
05210     _ValueType;
05211 
05212       // concept requirements
05213       __glibcxx_function_requires(_Mutable_RandomAccessIteratorConcept<
05214         _RandomAccessIterator>)
05215       __glibcxx_function_requires(_LessThanComparableConcept<_ValueType>)
05216       __glibcxx_requires_valid_range(__first, __last);
05217 
05218       if (__first != __last)
05219     {
05220       std::__introsort_loop(__first, __last,
05221                 std::__lg(__last - __first) * 2);
05222       std::__final_insertion_sort(__first, __last);
05223     }
05224     }
```

__lg函数是计算递归深度，用来控制分割恶化，当递归深度达到该值改用堆排序，因为堆排序是时间复杂度恒定为nlogn



```cpp
/// This is a helper function for the sort routines.  Precondition: __n > 0.
02308   template<typename _Size>
02309     inline _Size
02310     __lg(_Size __n)
02311     {
02312       _Size __k;
02313       for (__k = 0; __n != 0; __n >>= 1)
02314     ++__k;
02315       return __k - 1;
02316     }
```

快速排序的实现如下：



```cpp
02242   /// This is a helper function for the sort routine.
02243   template<typename _RandomAccessIterator, typename _Size>
02244     void
02245     __introsort_loop(_RandomAccessIterator __first,
02246              _RandomAccessIterator __last,
02247              _Size __depth_limit)
02248     {
02249       typedef typename iterator_traits<_RandomAccessIterator>::value_type
02250     _ValueType;
02251 
      //区间数目大于_S_threshold采用快速排序
02252       while (__last - __first > int(_S_threshold))
02253     {
           //达到指定递归深度，改用堆排序
02254       if (__depth_limit == 0)
02255         {
02256           _GLIBCXX_STD_P::partial_sort(__first, __last, __last);
02257           return;
02258         }
02259       --__depth_limit;
02260       _RandomAccessIterator __cut =
02261         std::__unguarded_partition(__first, __last,
02262                        _ValueType(std::__median(*__first,
02263                                 *(__first
02264                                   + (__last
02265                                      - __first)
02266                                   / 2),
02267                                 *(__last
02268                                   - 1))));
02269       std::__introsort_loop(__cut, __last, __depth_limit);
02270       __last = __cut;
02271     }
02272     }
```

插入排序部分的实现：



```cpp
/// This is a helper function for the sort routine.
02171   template<typename _RandomAccessIterator>
02172     void
02173     __final_insertion_sort(_RandomAccessIterator __first,
02174                _RandomAccessIterator __last)
02175     {
02176       if (__last - __first > int(_S_threshold))
02177     {
02178       std::__insertion_sort(__first, __first + int(_S_threshold));
02179       std::__unguarded_insertion_sort(__first + int(_S_threshold), __last);
02180     }
02181       else
02182     std::__insertion_sort(__first, __last);
02183     }

02093   /// This is a helper function for the sort routine.
02094   template<typename _RandomAccessIterator>
02095     void
02096     __insertion_sort(_RandomAccessIterator __first,
02097              _RandomAccessIterator __last)
02098     {
02099       if (__first == __last)
02100     return;
02101 
02102       for (_RandomAccessIterator __i = __first + 1; __i != __last; ++__i)
02103     {
02104       typename iterator_traits<_RandomAccessIterator>::value_type
02105         __val = *__i;
02106       if (__val < *__first)
02107         {
02108           std::copy_backward(__first, __i, __i + 1);
02109           *__first = __val;
02110         }
02111       else
02112         std::__unguarded_linear_insert(__i, __val);
02113     }
02114     }
```



![img](https:////upload-images.jianshu.io/upload_images/1791669-7e38bc5e1eb57cfa.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)


 图片来源：[C++一道深坑面试题：STL里sort算法用的是什么排序算法？]: [https://blog.csdn.net/qq_35440678/article/details/80147601](https://links.jianshu.com/go?to=https%3A%2F%2Fblog.csdn.net%2Fqq_35440678%2Farticle%2Fdetails%2F80147601)



## 26.4 资料

《算法》
 [冒泡排序和选择排序的区别] : [https://blog.csdn.net/weixin_41887155/article/details/85799820](https://links.jianshu.com/go?to=https%3A%2F%2Fblog.csdn.net%2Fweixin_41887155%2Farticle%2Fdetails%2F85799820)
 [排序算法详解（一）直接插入排序] :  [https://www.bilibili.com/video/BV1Jv41167eL?from=search&seid=385501809024223768](https://links.jianshu.com/go?to=https%3A%2F%2Fwww.bilibili.com%2Fvideo%2FBV1Jv41167eL%3Ffrom%3Dsearch%26seid%3D385501809024223768)

[C++一道深坑面试题：STL里sort算法用的是什么排序算法？]: https://blog.csdn.net/qq_35440678/article/details/80147601(https://links.jianshu.com/go?to=https%3A%2F%2Fblog.csdn.net%2Fqq_35440678%2Farticle%2Fdetails%2F80147601)

## 26.5 收获

1. 理解并实现了选择排序和插入排序
2. 了解stl中sort使用快速排序、堆排序、插入排序的原因以及代码实现
    排序算法还有很多其他类型，我们比如希尔排序、归并排序、桶排序等，我们这个阶段暂时不做学习实践，根据需有再续。