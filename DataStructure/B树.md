# B树简介

![image-20221205174238126](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221205174238126.png)

# m阶B树的性质

![image-20221205175203972](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221205175203972.png)

数据库实现中一般用几阶B树？

- 200 ~ 300

# B树 vs 二叉搜索树

**B树** 和 **二叉搜索树**，在逻辑上是等价的；

多代节点合并，可以获得一个超级节点

- 2 代合并的超级节点，最多拥有 4 个子节点（至少是 4 阶B树）
- 3 代合并的超级节点，最多拥有 8 个子节点（至少是 8 阶B树）
- n 代合并的超级节点，最多拥有 2n 个子节点（ 至少是 2n 阶B树）

m 阶 B树，最多需要 log2m 代合并；

![image-20221205182341195](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221205182341195.png)

# 搜索

![image-20221205182212393](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221205182212393.png)

# 添加 – 上溢

新添加的元素必定是添加到叶子节点：

![image-20221205194711019](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221205194711019.png)

插入 55：

![image-20221205194730587](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221205194730587.png)

插入 95：

![image-20221205194747482](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221205194747482.png)

再插入 98 呢？（假设这是一棵 4阶B树）

- 最右下角的叶子节点的元素个数将超过限制
- 这种现象可以称之为：**上溢（overflow）**

## 添加 – 上溢的解决(假设5阶)

上溢节点的元素个数必然等于 m；

假设上溢节点最中间元素的位置为 k

- 将 k 位置的元素向上与父节点合并

- 将 [0, k - 1] 和 [k + 1, m - 1] 位置的元素分裂成 2 个子节点
  这 2 个子节点的元素个数，必然都不会低于最低限制（┌ m/2 ┐ − 1）

一次分裂完毕后，有可能导致父节点上溢，依然按照上述方法解决

- 最极端的情况，有可能一直分裂到根节点

![image-20221205194857539](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221205194857539.png)

![image-20221205194907322](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221205194907322.png)

![image-20221205194917377](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221205194917377.png)

添加示例：

![image-20221205194939795](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221205194939795.png)

插入 98：

![image-20221205195021538](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221205195021538.png)

插入 52：

![image-20221205195049842](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221205195049842.png)

插入 54：

![image-20221205195104737](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221205195104737.png)

# 删除

## 删除 – 叶子节点

![image-20221205195513658](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221205195513658.png)

## 删除 – 非叶子节点

![image-20221205195602259](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221205195602259.png)

## 删除 – 下溢

![image-20221205195733218](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221205195733218.png)

## 删除 – 下溢的解决

![image-20221205220250600](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221205220250600.png)



# 4阶B树

如果先学习 4 阶 B 树（2 - 3 - 4 树），将能更好地学习理解红黑树：

4阶B树的性质：

- 所有节点能存储的元素个数 x ：1 ≤ x ≤ 3
- 所有**非叶子节点**的子节点个数 y ：2 ≤ y ≤ 4

添加：从 1 添加到 22

![image-20221205221427235](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221205221427235.png)