### 前置知识

前序遍历(pre-order)：根-左-右

```python
def preorder(self,root):
	if root:
		self.traverse_path.append(root.val)
		self.preorder(root.left)
		self.preorder(root.right)
```

中序遍历(in-order)：左-根-右

```python
def inorder(self,root):
	if root:
		self.inorder(root.left)
		self.traverse_path.append(root.val)
		self.inorder(root.right)
```

后序遍历(post-order)：左-右-根

```python
def postorder(self,root):
  if root:
    self.postorder(root.left)
    self.postorder(root.right)
    self.traverse_path.append(root.val)
```

### AVL树

树的查询复杂度等于树的深度，AVL树是平衡二叉树

平衡因子绝对值 <= 1

四种旋转操作

### 红黑树

红黑树是近似平衡二叉树，它能确保任何一个节点的左右子树的高度差小于两倍。

每个节点要么是红色，要么是黑色

根节点是黑色

每个叶节点（NIL节点，空节点）是黑色的

不能有相邻接的两个红色节点

从任一节点到其每个叶子节点的所有路径都包含相同数目的黑色节点

