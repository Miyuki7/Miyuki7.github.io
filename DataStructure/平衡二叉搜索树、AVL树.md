# 二叉搜索树缺点分析

AVL树是在二叉搜索树 的基础上学习的。

![image-20221202180907624](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221202180907624.png)

* 当 n 比较大时，两者的性能差异比较大
* 比如 n = 1000000 时，二叉搜索树的最低高度是 20，最高高度是 1000000；

由此可见，二叉搜索树添加节点时可能会导致二叉搜索树退化成链表；
而删除节点时也可能会导致二叉搜索树退化成链表；

# 改进二叉搜索树

![image-20221202181356246](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221202181356246.png)

![image-20221202181557287](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221202181557287.png)

# AVL树