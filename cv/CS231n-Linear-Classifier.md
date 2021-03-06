# Intro. 线性分类

线性分类比kNN更强, 而且也能更自然地推广到神经网络和卷积神经网络.

线性分类包括两个部分:

* **打分函数(Score function):** 把图片像素映射到类别分数.
* **损失函数(Loss function):** 量化衡量预测分类和答案的吻合度(agreement).

按照上述两部分看待线性分类模型, 就能把线性分类转化成数学上的最优化问题. 这里, 我们把打分函数的参数看成自变量, 找到使损失函数最小的参数取值.

# 1. 打分函数

## 1.1. 从像素到打分的参数化映射

规定:

* $$x_i$$: 第$$i$$个训练样本的(列)向量. 可能会问, 图片是二维的, 怎么变成一维? -- 把它展平(flatten)就变成一维的了.
  * $$x_i\in \mathbb{R}^D$$. 其中$$D$$是图片的像素数.
  * $$i=1..N$$. 其中$$N$$是训练样本数.
* $$y_i$$: 第$$i$$个训练样本的标签.
  * $$y_i=1..K$$. 其中$$K$$是类别数.
* $$f: \mathbb{R}^D\mapsto \mathbb{R}^K$$: 打分函数.

-----

**定义: 线性分类器**
$$
\forall x_i \in T, f\left(x_i\right)=Wx_i+b
$$
其中:

* $$x_i$$: 训练集$$T$$中的第$$i$$个样本. 已经被展平成$$1\times D$$的列向量.


* $$W$$: 权重矩阵, 形状为$$K\times D$$.
* $$b$$: 截距(bias), 也叫偏移量. 形状为$$1\times K$$.

-----

注意:

* 矩阵$$W$$的每一行都是一个分类器.
* 训练完以后, 知识记录在$$W$$和$$b$$中, 因此训练完后就不再需要训练集. 这点比kNN先进很多.

## 1.2. 权重矩阵W的诠释

因为输入的是颜色值, 所以这个映射函数实际上学到的是**喜欢/不喜欢某颜色出现在某位置**.

例如: 由于轮船通常出现在海上, 可以认为轮船的背景是大片蓝色的. 因此我们可以猜想, 权重矩阵W在"轮船"那一行上, 一定在蓝色通道上有很多正值, 而在其他通道上有很多负值.



----

**图1: 把图像映射到分数**

![img](https://cs231n.github.io/assets/imagemap.jpg)

为了视觉上的简便和直观, 我们假设

- 输入图像只有2x2=4个像素
- 输入图像是灰度图

注意图中的权重W并不好, 它把一只猫分成了狗.

-----

### 1.2.1. 用线性规划来解释

----

**图2: 每张训练图像被当成高维空间的点, 而权重矩阵的每一行都成为分类超平面**

实际上, W的每一行都是超平面的**法向量**(normal). 如果点出现在法向量所指的一侧(和法向量点乘为正) 则判定它是这一类的.

![img](https://cs231n.github.io/assets/pixelspace.jpeg)

----

### 1.2.2. 用模板匹配来解释

$$W$$的每一行实际上还可以理解成**模板**(template). 通过计算模板和图片的点乘, 得到图片在每一类上的得分.

笔者注: 这个想法和Dynamic Routing Between Capsules中的内积是相同的, 都是用内积来衡量相似度.

----

**图3: 线性分类器在CIFAR-10上学到的模板**

以ship为例, 和我们预想的一样, ship的模板有大量的蓝色.

![img](https://cs231n.github.io/assets/templates.jpg)

----

#### 特征合并现象

需要特别注意的是, 线性分类器有**特征合并**的现象.

以horse为例, 可以发现, 这是个双头马. 这是因为分类器学到了两种不同朝向的马, 并把这两种朝向的马叠加到一个模板里.

再以car为例, car的模板看起来叠加了各种朝向\各种颜色的车. 而最终的car模板看起来偏红, 这也说明CIFAR-10上红色的车偏多.

car的例子也提示我们, 线性分类器在学习不同颜色的车上表现很差.

## 1.3. 去掉讨厌的截距项 

给输入$$x_i$$追加一维, 这一维的值恒为1. 给权重矩阵追加一列, 这一列就是$$b$$. 这一块比较简单, 不记载了. 详见[CS231n - Linear Classifier](https://cs231n.github.io/linear-classify/#interpret)的Bias Trick部分.

# 2. 损失函数

接下来会讲到两个老生常谈的模型: SVM和Softmax. 实际上, 它们的前向是完全相同的. 不同之处在于它们的损失函数.

## 2.1. SVM与margin loss.

----

**图4: 多类SVM希望, 对于每一个训练样本, 正确类的得分比所有其他错误类的得分至少高出$$\Delta$$**

![img](https://cs231n.github.io/assets/margin.jpg)

-----

规定:

* $$s_i$$: 第$$i$$个训练样本的得分. 即$$s_i=f\left(x_i\right\vert W)$$.
  * $$s_{i,j}$$: 第$$i$$个训练样本在第$$j$$类的得分.
* $$L_i$$: 第$$i$$个训练样本的分类损失.
* $$L\left(W\right)$$: 分类器的总损失.

### 2.1.1. 每个样本的分类损失

$$
L_i=\sum_{j\neq y_i}{\max\left(0, s_{i,j}-s_{i,y_i}+\Delta\right)}
$$

我在看的时候产生过这些问题:

* Q: 为什么要用错误分类的分数减去正确分类的分数, 而不是反过来?
* A: 因为这是个损失函数, 要保证错误分类的分数越高, 损失越大.
* Q: 为什么要取max(0, -)?
* A: 因为如果$$s_{i,j}-s_{i,y_i}+\Delta<0$$, 则说明正确类的分数已经第j类的分数大$$\Delta$$.

根据上面的思考, 可以给出训练集上的分类损失, 也叫作数据损失(data loss). 数据损失是样本损失的算术平均值:
$$
L_\mathrm{data}=\dfrac{1}{N}\sum_{i=1}^{N}{L_i}
$$

### 2.1.2. 损失正则化

只使用分类损失有个问题, 就是矩阵W不唯一. 两种较简单地生成这种矩阵的方法为:

1) 给矩阵乘以一个大于1的常数$$\lambda$$. 这样, 所有分类分数都变成原来的$$\lambda$$倍, 对分类结果不产生影响.

2) 对于某个输入$$x_i=[1,1,1,1]^T$$, 以及$$W$$的某两行$$w_1=[1,0,0,0]$$和$$w_2=[0.25, 0.25, 0.25, 0.25]$$. 显然, $$w_1 \cdot x_i = w_2 \cdot x_i = 1$$. 这会导致分类结果产生歧义.

对于第二种情况, $$w_1$$和$$w_2$$本来是不同的模板, 却和$$x_i$$有着相同的相似度, 这不是我们所希望的. 我们需要找到一种方式区分开.

一种简单的方式就是计算$$W$$的元素的平方和, 这也就是常说的L2-loss. 在第二种情况下, $$w_1$$产生的L2-loss为1, 而$$w_2$$产生的L2-loss为0.25. 

根据上面的思考, 可以给出正则化损失(regularization loss)
$$
L_\mathrm{reg}=\sum_{k=1}^{K}\sum_{d=1}^{D+1}{W_{k,d}^2}
$$

### 2.1.3. 总的损失

$$
L(W)=L_\mathrm{data}+\lambda L_\mathrm{reg}
$$

### 2.1.4. 超参的设置

损失函数中有两个超参, $$\Delta$$和$$\lambda$$.

实际上, $$\Delta$$在任何情况都可以设为1. 原因如下:

别看$$\Delta$$和$$\lambda$$是两个参数, 实际上它俩控制着同一件事, 那就是平衡Data loss和Regularization loss. 而在学习的过程中, $$\Delta$$跟W成正比.

## 2.2. Cross-entropy loss与Softmax

Softmax的前向和SVM的前向一样, 仍然是上面讲过的线性打分函数. 

和SVM不同的是, 它的损失函数是**交叉熵损失**(cross entropy loss). 每一个训练样本$$x_i$$的交叉熵损失为:
$$
L_i =-\log\left(\dfrac{\exp\left(f_{y_i}(x_i)\right)}{\sum_j{\exp\left(f_j(x_i)\right)}}\right)
$$
它的等价形式为:
$$
L_i =-f_{y_i}(x_i)+\log\sum_j{\exp\left(f_j(x_i)\right)}
$$
其中, $$f_j\left(x_i\right)$$是输出向量$$f\left(x_i\right)$$的第$$j$$个元素.

除了每个训练样本的损失和SVM不同, 其他的损失是相同的. 也就是说, Softmax损失函数**也分为数据损失和正则损失两大项**. 形式如公式(3)~(5)所示.

### 2.2.1. 信息熵



