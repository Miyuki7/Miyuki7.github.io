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

![image-20221204151017801](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221204151017801.png)

## BST 对比 AVLTree

![image-20221204151236009](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221204151236009.png)

## 体系结构

![image-20221204151208243](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221204151208243.png)



## 添加节点导致失衡

![image-20221204155504424](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221204155504424.png)

### LL – 右旋转（单旋）

让 `p` 成为这颗子树的根节点

- `g.left = p.right`
- `p.right = g`

旋转后仍然是一颗 二叉搜索树：`T0 < n < T1 < p < T2 < g < T3`

![image-20221204160504048](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221204160504048.png)

``` java
/**
 * 右旋转
 */
private void rotateRight(Node<E> grand) {
    Node<E> parent = grand.left;
    Node<E> child = parent.right;
    grand.left = child;
    parent.right = grand;

    afterRotate(grand, parent, child);
}
```



还需要注意维护的内容

- `T2`、`p`、`g` 的 `parent` 属性
- 先后更新 `g`、`p` 的高度

### RR – 左旋转（单旋）

让 `p` 成为这颗子树的根节点

- `g.right = p.left`
- `p.left = g`

旋转后仍然是一颗 二叉搜索树：`T0 < g < T1 < o < T2 < n < T3`

![image-20221204161809497](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221204161809497.png)

还需要注意维护的内容

- `T1`、`p`、`g` 的 `parent` 属性
- 先后更新 `g`、`p` 的高度

``` java
/**
 * 左旋转
 */
private void rotateLeft(Node<E> grand) {
	Node<E> parent = grand.right;
	Node<E> child = parent.left;
	grand.right = child;
	parent.left = grand;
	
	afterRotate(grand, parent, child);
}
```

### LR – 先左旋，再右旋（双旋）

LR 就是 先进行 **左旋转 – RR**、再进行 **右旋转 – LL**；

* 先左旋转：p.right = n.left、n.left = p
* 再右旋转：g.left = n.right、n.right = g

旋转后仍然是一颗 二叉搜索树：T0 < p < T1 < n < T2 < g < T3
![image-20221204165132404](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221204165132404.png)

### RL – 先右旋，再左旋（双旋）

RL 就是 先进行 **右旋转 – LL**、再进行 **左旋转 – RR**；

* 先右旋转：p.left = n.right、n.right = p
* 再左旋转：g.right = n.left、n.left = g

旋转后仍然是一颗 二叉搜索树：T0 < g < T1 < n < T2 < p < T3

![image-20221204162332458](C:\Users\14266\AppData\Roaming\Typora\typora-user-images\image-20221204162332458.png)

### 旋转之后维护的内容

``` java
/**
 * 公共代码：不管是左旋转、右旋转，都要执行的
 * @param grand 失衡节点
 * @param parent 失衡节点的tallerChild
 * @param child child g和p需要交换的子树(本来是p的子树, 后面会变成g的子树)
 */
private void afterRotate(Node<E> grand, Node<E> parent, Node<E> child) {
    // 让parent成为子树的根节点
    parent.parent = grand.parent;
    if (grand.isLeftChild()) {
        grand.parent.left = parent;
    } else if (grand.isRightChild()) {
        grand.parent.right = parent;
    } else {// grand是root节点
        root = parent;
    }

    // 更新child的parent
    if (child != null) {
        child.parent = grand;
    }

    // 更新grand的parent
    grand.parent = parent;

    // 更新高度
    updateHeight(grand);
    updateHeight(parent);
}
```

### 添加之后的修复图解

输入数据：13, 14, 15, 12, 11, 17, 16, 8, 9, 1

输入13：正常
输入14：正常
输入15：13失衡，RR，左旋转

![image-20221204221312697](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221204221312697.png)

输入12：正常
输入11：13失衡，LL，右旋转

![image-20221204221331109](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221204221331109.png)

输入17：正常
输入16：15失衡，RL，先右选择、再左旋转

![image-20221204221351854](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221204221351854.png)

输入8：正常
输入9：11失衡，LR，先左旋转、再右旋转

![image-20221204221410922](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221204221410922.png)

输入1：12失衡，LL，右旋转

![image-20221204221436206](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221204221436206.png)

### 添加之后的修复 - 代码实现

``` java
/**
 * 增加节点后的调整
 */
@Override
protected void afterAdd(Node<E> node) {
    while ((node = node.parent) != null) {
        if (isBalanced(node)) { // 如果平衡
            // 更新高度
            updateHeight(node);
        } else { // 如果不平衡
            // 恢复平衡
            rebalance(node);
            // 只要恢复了最下面的子树的平衡, 则整棵树恢复平衡
            break;
        }
    }
}
```

``` java
/**
 * 恢复平衡
 * @param grand 高度最低的那个不平衡节点
 */
private void rebalance(Node<E> grand) {
    Node<E> parent = ((AVLNode<E>) grand).tallerChild();
    Node<E> node = ((AVLNode<E>) parent).tallerChild();
    if (parent.isLeftChild()) {//L
        if (node.isLeftChild()) {//LL
            rotateRight(grand);//LL则右旋
        } else {//LR
            rotateLeft(parent);
            rotateRight(grand);
        }
    } else {//R
        if (node.isLeftChild()) {//RL
            rotateRight(parent);
            rotateLeft(grand);
        } else {//RR
            rotateLeft(grand);//RR则左旋
        }
    }
}
```

### 统一所有的旋转操作

![image-20221204221533792](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221204221533792.png)

``` java
/**
 * 统一旋转
 */
private void rotate(
    Node<E> r, // 子树的根节点
    Node<E> b, Node<E> c,
    Node<E> d,
    Node<E> e, Node<E> f) {
    // 让d成为这颗子树的根结点
    d.parent = r.parent;
    if (r.isLeftChild()) {
        r.parent.left = d;
    } else if (r.isRightChild()) {
        r.parent.right = d;
    } else {
        root = d;
    }
    // b-c
    b.right = c;
    if (c != null) {
        c.parent = b;
    }
    updateHeight(b);

    // e-f
    f.left = e;
    if (e != null) {
        e.parent = f;
    }
    updateHeight(f);

    // b-d-f
    d.left = b;
    d.right = f;
    b.parent = d;
    f.parent = d;
    updateHeight(d);
}
```

``` java
private void rebalance(Node<E> grand) {
    Node<E> parent = ((AVLNode<E>) grand).tallerChild();
    Node<E> node = ((AVLNode<E>) parent).tallerChild();
    if (parent.isLeftChild()) {//L
        if (node.isLeftChild()) {//LL
            rotate(grand, node, node.right, parent, parent.right, grand);
        } else {//LR
            rotate(grand, parent, node.left, node, node.right, grand);
        }
    } else {//R
        if (node.isLeftChild()) {//RL
            rotate(grand, grand, node.left, node, node.right, parent);
        } else {//RR
            rotate(grand, grand, parent.left, parent, node.left, node);
        }
    }
}
```

### 删除节点导致的失衡

示例：删除子树中的 16

- 可能会导致**父节点**或**祖先节点**失衡（只有1个节点会失衡），其他节点，都不可能失衡(下图中有误。这个描述是正确的)

![image-20221204221715040](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221204221715040.png)

### LL – 右旋转（单旋）

- 如果绿色节点不存在，更高层的祖先节点可能也会失衡，需要再次恢复平衡，然后又可能导致更高层的祖先节点失衡…
- 极端情况下，所有祖先节点都需要进行恢复平衡的操作，共 O(logn) 次调整

![image-20221204230309912](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221204230309912.png)

### RR – 左旋转（单旋）

![image-20221204230325095](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221204230325095.png)

### LR – RR左旋转，LL右旋转（双旋）

![image-20221204230343310](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221204230343310.png)

### RL – LL右旋转，RR左旋转（双旋）

![image-20221204230401546](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221204230401546.png)

## AVL树总结

![image-20221204232241807](https://cdn.jsdelivr.net/gh/Miyuki7/image-host/blog-imgimage-20221204232241807.png)



AVL树代码

``` java

import java.util.Comparator;

public class AVLTree<E> extends BST<E> {

    public AVLTree(Comparator<E> comparator) {
        super(comparator);
    }

    public AVLTree() {
        this(null);
    }

    // AVL树的节点，需要计算平衡因子，因此比普通二叉树多维护一个height属性(将height放入普通二叉树里没有用处，浪费空间)
    private static class AVLNode<E> extends Node<E> {
        int height = 1;

        public AVLNode(E element, Node<E> parent) {
            super(element, parent);
        }

        public int balanceFactor() { // 获取该节点平衡因子(左子树高度 - 右子树高度)
            int leftHeight = left == null ? 0 : ((AVLNode<E>) left).height;
            int rightHeight = right == null ? 0 : ((AVLNode<E>) right).height;
            return leftHeight - rightHeight;
        }

        public void updateHeight() { // 更新高度
            int leftHeight = left == null ? 0 : ((AVLNode<E>) left).height;
            int rightHeight = right == null ? 0 : ((AVLNode<E>) right).height;
            height = 1 + Math.max(leftHeight, rightHeight);
        }

        public Node<E> tallerChild() {
            int leftHeight = left == null ? 0 : ((AVLNode<E>) left).height;
            int rightHeight = right == null ? 0 : ((AVLNode<E>) right).height;
            if (leftHeight > rightHeight) return left;
            if (rightHeight > leftHeight) return right;
            // 高度一样则返回同方向的，左子节点则返回左，否则返回右
            return isLeftChild() ? left : right;
        }

        @Override
        public String toString() {
            String parentString = "null";
            if (parent != null) {
                parentString = parent.element.toString();
            }
            return element + "_p(" + parentString + ")_h(" + height + ")";
        }
    }

    /**
     * 增加节点后的调整
     */
    @Override
    protected void afterAdd(Node<E> node) {
        while ((node = node.parent) != null) {
            if (isBalanced(node)) { // 如果平衡
                // 更新高度
                updateHeight(node);
            } else { // 如果不平衡
                // 恢复平衡
                rebalance(node);
                // AVL树中, 只要恢复了最下面的子树的平衡, 则整棵树恢复平衡
                break;
            }
        }
    }

    /**
     * 删除节点后的调整
     */
    @Override
    protected void afterRemove(Node<E> node) {
        while ((node = node.parent) != null) {
            if (isBalanced(node)) {
                // 更新高度
                updateHeight(node);
            } else {
                // 恢复平衡
                rebalance(node);
            }
        }
    }

    /**
     * 重写父类中的 createNode
     * 返回 AVLNode
     */
    @Override
    protected Node<E> createNode(E element, Node<E> parent) {
        return new AVLNode<>(element, parent);
    }

    /**
     * 判断传入节点是否平衡（平衡因子的绝对值 <= 1）
     */
    private boolean isBalanced(Node<E> node) {
        return Math.abs(((AVLNode<E>) node).balanceFactor()) <= 1;
    }

    /**
     * 更新高度
     */
    private void updateHeight(Node<E> node) {
        ((AVLNode<E>) node).updateHeight();
    }

    /**
     * 恢复平衡
     * @param grand 高度最低的那个不平衡节点
     */
    private void rebalance2(Node<E> grand) {
        Node<E> parent = ((AVLNode<E>) grand).tallerChild();
        Node<E> node = ((AVLNode<E>) parent).tallerChild();
        if (parent.isLeftChild()) { // L
            if (node.isLeftChild()) { // LL
                rotateRight(grand); // LL则右旋
            } else { // LR
                rotateLeft(parent);
                rotateRight(grand);
            }
        } else { // R
            if (node.isLeftChild()) { // RL
                rotateRight(parent);
                rotateLeft(grand);
            } else { // RR
                rotateLeft(grand); // RR则左旋
            }
        }
    }

    private void rebalance(Node<E> grand) {
        Node<E> parent = ((AVLNode<E>) grand).tallerChild();
        Node<E> node = ((AVLNode<E>) parent).tallerChild();
        if (parent.isLeftChild()) {//L
            if (node.isLeftChild()) {//LL
                rotate(grand, node, node.right, parent, parent.right, grand);
            } else {//LR
                rotate(grand, parent, node.left, node, node.right, grand);
            }
        } else {//R
            if (node.isLeftChild()) {//RL
                rotate(grand, grand, node.left, node, node.right, parent);
            } else {//RR
                rotate(grand, grand, parent.left, parent, node.left, node);
            }
        }
    }

    /**
     * 统一旋转
     */
    private void rotate(
            Node<E> r, // 子树的根节点
            Node<E> b, Node<E> c,
            Node<E> d,
            Node<E> e, Node<E> f) {
        // 让d成为这颗子树的根结点
        d.parent = r.parent;
        if (r.isLeftChild()) {
            r.parent.left = d;
        } else if (r.isRightChild()) {
            r.parent.right = d;
        } else {
            root = d;
        }
        // b-c
        b.right = c;
        if (c != null) {
            c.parent = b;
        }
        updateHeight(b);

        // e-f
        f.left = e;
        if (e != null) {
            e.parent = f;
        }
        updateHeight(f);

        // b-d-f
        d.left = b;
        d.right = f;
        b.parent = d;
        f.parent = d;
        updateHeight(d);
    }
	/*private void rotate(
			Node<E> r, // 子树的根节点
			Node<E> a, Node<E> b, Node<E> c,
			Node<E> d,
			Node<E> e, Node<E> f, Node<E> g) {
		// 让d成为这颗子树的根结点
		d.parent = r.parent;
		if(r.isLeftChild()){
			r.parent.left = d;
		}else if(r.isRightChild()){
			r.parent.right = d;
		}else{
			root = d;
		}
		// a-b-c
		b.left = a;
		if(a!=null){
			a.parent = b;
		}
		b.right = c;
		if(c!=null){
			c.parent = b;
		}
		updateHeight(b);
		
		// e-f-g
		f.left = e;
		if(e != null){
			e.parent = f;
		}
		f.right = g;
		if(g != null){
			g.parent = f;
		}
		updateHeight(f);
		
		// b-d-f
		d.left = b;
		d.right = f;
		b.parent = d;
		f.parent = d;
		updateHeight(d);
	}*/

    /**
     * 左旋转
     */
    private void rotateLeft(Node<E> grand) {
        Node<E> parent = grand.right;
        Node<E> child = parent.left;
        grand.right = child;
        parent.left = grand;

        afterRotate(grand, parent, child);
    }

    /**
     * 右旋转
     */
    private void rotateRight(Node<E> grand) {
        Node<E> parent = grand.left;
        Node<E> child = parent.right;
        grand.left = child;
        parent.right = grand;

        afterRotate(grand, parent, child);
    }

    /**
     * 公共代码：不管是左旋转、右旋转，都要执行的
     * @param grand 失衡节点
     * @param parent 失衡节点的tallerChild
     * @param child child g和p需要交换的子树(本来是p的子树, 后面会变成g的子树)
     */
    private void afterRotate(Node<E> grand, Node<E> parent, Node<E> child) {
        // 让parent成为子树的根节点
        parent.parent = grand.parent;
        if (grand.isLeftChild()) {
            grand.parent.left = parent;
        } else if (grand.isRightChild()) {
            grand.parent.right = parent;
        } else {// grand是root节点
            root = parent;
        }

        // 更新child的parent
        if (child != null) {
            child.parent = grand;
        }

        // 更新grand的parent
        grand.parent = parent;

        // 更新高度
        updateHeight(grand);
        updateHeight(parent);
    }

}
```

