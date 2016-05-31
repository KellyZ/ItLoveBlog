##  选择排序（冒泡排序）

每次都从剩余的数据中选择出最大（或最小）放到已排序数据的后面；

缺点：每次都需要将所有**未排序**的数据相互比较得到最大（或最小）的数据，增加了比较次数；

时间复杂度：O(n^2)

## 插入排序

每次都从剩余的数据中选择一个（比如第一个）插入到已排序数据中某个正确的位置；

优点：剩余**未排序**数据间不再相互比较，而是拿未排序的某个数据和**已排序**数据进行比较，插入到正确的位置，很显然一旦找到正确位置将不再需要继续比较；

缺点：假如某个数据相对大于（或小于）已排序的数据，那么这个数据将可能要和所有已排序的数据进行比较，一步步的插入到最前面（或最后面）

## 希尔排序

上述插入排序中需要从**已排序**中一步步找到正确位置，而随机性决定了这里面有相当多的“多余”的比较交换动作，既然随机性：那就将整个随机序列按间隔为h的元素形成子序列（共分割成h个子序列，每个子序列最多N/h+1个元素），然后对每个子序列进行**插入排序**：

    for (int i=h; i<N; i++)  //h个子序列一起插入排序，h~2h是h个子序列的第二个元素
    {   //将a[i]插入到a[i-h], a[i-2h], a[i-3h]...这个已排序的序列中，注入是插入
        for (int j=i; j>=h && less(a[j], a[j-h]); j -=h )
        {
            exch(a, j, j-h);
        }
    }

然后不断减少h，直至h为1，及整个序列插入排序（如果是从小到大排列，此时大部分小数据在前面，大部分大数据在后面）

那么h初始为多少？以及h的递减规则如何？哪种是最优的？目前无法证明哪个是“最优的”。

比如h可以按下生1, 4, 13, 40... 这样的序列：

    while (h < N/3 ) h=3*h + 1;

优点：将位置不正确的尽快的移动到靠近正确的位置。

## 归并排序

对无序序列的左右两个子序列分别排序，然后将两个子序列归并，两个子序列可以继续这样递归操作进行排序，如此，就好像颗倒立的树一样，不断的拆分排序然后归并；

    public static void merge(Comparable[] a, int lo, int mid, int hi)
    {
        int i =lo, j=mid+1;

        //复制一份
        for（int k = lo; k<=hi; k++)
        {
            aux[k] = a[k];
        }
        // 归并回到a[lo...hi]
        for (int k=lo; k<= hi; k++){
            if (i>mid)       a[k] = aux[j++];
            else if (j>hi)   a[k] = aux[i++];
            else if( less(aux[j], aux[i]) )  a[k] = aux[j++];
            else  a[k] = aux[i++];
       }
    }

## 快速排序

归并排序默认是对半分，快速排序是随机切分，关键在切分：一般的策略是先随意选取a[lo]作为切分元素，然后左右扫描交换，使得切分位置所有左边元素都小于切分元素，右边元素都大于切分元素。

![快速排序](https://raw.githubusercontent.com/KellyZ/ItLoveBlog/master/images/Visual-and-intuitive-feel-of-7-common-sorting-algorithms.gif)

    private static int partition(Comparable[] a, int lo, int hi)
    {
        int i=lo, j=hi+1;
        Comparable v = a[lo];
        while (true)
        {
            //扫描左右，检查扫描是否结束并交换元素
            while (less(a[++i], v))     if (i==hi) break;
            while (less(a[v, a[--j]))   if (j==lo) break;
            if (i>=j) break;
            exch(a, i, j);
        }
        exch(a,lo, j);
        return j;
    }

## 堆排序

堆（二叉堆）可以视为一棵完全的二叉树，完全二叉树的一个“优秀”的性质是，除了最底层之外，每一层都是满的，这使得堆可以利用数组来表示（普通的一般的二叉树通常用链表作为基本容器表示），每一个结点对应数组中的一个元素。

![堆排序](https://raw.githubusercontent.com/KellyZ/ItLoveBlog/master/images/heap-and-array.png)

对于给定的某个结点的下标 i，可以很容易的计算出这个结点的父结点、孩子结点的下标：

1. Parent(i) = floor(i/2)，i 的父节点下标
2. Left(i) = 2i，i 的左子节点下标
3. Right(i) = 2i + 1，i 的右子节点下标



## 排序算法的选择

1.数据规模较小

（1）待排序列基本序的情况下，可以选择直接插入排序；

（2）对稳定性不作要求宜用简单选择排序，对稳定性有要求宜用插入或冒泡

2.数据规模不是很大

（1）完全可以用内存空间，序列杂乱无序，对稳定性没有要求，快速排序，此时要付出 log（N）的额外空间。

（2）序列本身可能有序，对稳定性有要求，空间允许下，宜用**归并排序**

3.数据规模很大

（1）对稳定性有求，则可考虑归并排序。

（2）对稳定性没要求，宜用堆排序

4.序列初始基本有序（正序），宜用直接插入，冒泡

## 参考

1. http://jsrun.it/norahiko/oxIy  （排序可视化）
2. http://www.sorting-algorithms.com/  （排序可视化）
3. http://www.vaikan.com/sort-dance/  （排序舞蹈化）
4. http://www.jianshu.com/p/f5baf7f27a7e
5. http://wiki.jikexueyuan.com/project/java-special-topic/sort.html
6. http://bubkoo.com/2014/01/14/sort-algorithm/heap-sort/
7. http://zh.visualgo.net/
8. http://v.youku.com/v_show/id_XNjIwNTEzMTA0.html