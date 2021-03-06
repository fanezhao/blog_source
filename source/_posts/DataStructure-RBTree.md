---
title: 红黑树
date: 2020-06-05 01:00:00
tags:
 - 数据结构
---

## 红黑树

**什么是红黑树**？

红黑树，Red-Black Tree 「RBT」是一个自平衡(不是绝对的平衡)的二叉查找树(BST)。红黑树的五大性质如下：

- 1、节点是红色或者黑色；
- 2、根节点是黑色；
- 3、叶子节点是黑色；
- 4、红色节点必须具有两个黑色子节点；
- 5、从任一节点到其后代的叶子节点路径中包含相同个数的黑色节点。(黑高、黑色完美平衡)

{% qnimg 红黑树节点名称约定.jpg %}

## 红黑树的插入流程

红黑树的新节点都为红色（**红插**），因为根据上述的五大性质可知，插入节点为黑色的话一定会触发自平衡，如果是红色节点的话有可能触发自平衡。

红黑树的插入流程分为两步：

- 1、查找插入位置并插入。
- 2、进行自平衡操作。

### 查找插入位置

红黑树的查找根二叉查找树一样，根据节点对比向右或向左查找。

### 自平衡操作

红黑树的自平衡操作有三种：

#### 一、变色

结点颜色由黑变红或由红变黑。

#### 二、左旋

{% qnimg RBTree-LeftRotate.webp %}

- 1、以某个节点为支点进行旋转。
- 2、其右子节点变为旋转节点的父节点。
- 3、其右子节点的左子节点变为旋转节点的右子节点。

#### 三、右旋

{% qnimg RBTree-RightRotate.webp %}

- 1、以某个节点为支点进行旋转
- 2、其左子节点变为旋转节点的父节点
- 3、其左子节点的右子节点变为旋转节点的左子节点

## 红黑树插入情景分析

### 一、红黑树为空树

直接把插入节点作为根节点，并设为黑色即可。

### 二、插入节点Key已存在

更新当前节点的值为插入节点的值即可。

### 三、插入节点的父节点为黑色

找到父节点，直接插入即可，不影响黑色完美平衡。

{% qnimg 插入切点父节点为黑色.jpg %}

### 四、插入节点的叔叔节点存在且为红

{% qnimg 插入节点的叔叔节点存在且为红.jpg %}

- 1、其父节点和叔叔节点变黑。
- 2、祖父节点变红。
- 3、以祖父节点为当前节点，进行后续处理。

可以看到，如果我们把PP节点设为红色，如果PP节点的父节点是黑色，那么就无需做任何处理。但如果PP节点的父节点是红色，则违反了红黑树的性质了。所以需要将PP节点设置为当前节点，继续做自平衡处理，直到平衡为止。

### 五、叔叔节点不存在或为黑，且插入节点的父节点是祖父节点的左子节点

#### 1、插入节点是父节点的左子节点（LL双红）

{% qnimg LL双红.jpg %}

- 1、其父节点变黑、祖父节点变红。
- 2、右旋祖父节点。

#### 2、插入节点是父节点的右子节点（LR双红）

{% qnimg LR双红.jpg %}

- 1、其父节点左旋形成LL双红。
- 2、进行LL双红自平衡操作。

### 六、叔叔节点不存在或为黑，且插入节点的父节点是祖父节点的右子节点

#### 1、插入节点是父节点的右子节点（RR双红）

{% qnimg RR双红.jpg %}

- 1、其父节点变黑、祖父节点变红。
- 2、左旋祖父节点。

#### 2、插入节点是父节点的右子节点（LR双红）

{% qnimg RL双红.jpg %}

- 1、其父节点右旋形成RR双红。
- 2、进行RR双红自平衡操作。