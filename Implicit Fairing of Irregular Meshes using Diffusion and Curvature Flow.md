## Implicit Fairing of Irregular Meshes using Diffusion and Curvature Flow

*使用分布和曲率流隐式平滑不规则网格*



##### 主要目标：

当保留需要的几何特征时去除不需要的噪点和不平整的边界。

![](https://github.com/freyakniglty/algorithm/blob/master/images/1.png)



##### 本文介绍的方法有三个重要特性：

- implicit integration（隐式积分）
- 依靠测度的拉普拉斯算子提升扩散过程
- 鲁棒性的曲率流算子来平滑形状，不同于任何参数化

### 简介

**前人工作：**

Taubin 建立他的方法基于定义一个合适的对于任意连接性网格的频率的总结。

Kobbelt 考虑对于拉普拉斯在构造 interpolatory subdivision schemes相似离散的近似。



### **Implicit fairing：**

**一些定义：**

$(\lambda + \mu)L-\lambda\mu L^2$ 可以提供一个高斯滤波器最小化褶皱。其中$\lambda$ 和 $\mu$ 是用户设定的常量，用以保存没有被缩掉的属性。把这个叫做 $\lambda | \mu$ 算法。

其中 $L(X) = X_{uu}+X_{vv}$

$L^2(X)=L\circ L(X) = X_{uuuu} + 2X_{uuvv}+X_{vvvv}$



**分布等式：**

在网格上使噪声变少，可以通过*diffusion process*(扩散过程):

$\frac{\partial X}{\partial t} = \lambda L(X)$

其中拉普拉斯算子可以被线性地被umbrella operator(伞算子)近似，写成：

$L(x_i) = \frac{1}{m} \sum_{j\in N_1(i)}x_j-x_i$

${x_j}$ 是 $x_i$ 的邻域，m = #N1(i) 是领域的编号。

通过以上等式，可以将一个小的干扰施加在邻域，光滑较高的频率，但是主要的形状只有微小的变化。

一系列的网格可以被对扩散等式进行积分构造，则有：

$X^{n+1} = (I+\lambda dt L)X^n$

其中为了稳定，$\lambda dt <1$ 如果这个不等式不满足则会出现裂缝



**隐式积分：**

*forward Euler method* 是说以上的方法

主要想法是：如果我们使用新的网格来近似倒数，我们可以更快得到PDE的平衡态。

现在的积分是：$X^{n+1} = (I+\lambda dt L)X^n$

如果用隐式积分，则叫做 *backward Euler method* ，则转变为求解：

$(I-\lambda dtL)X^{n+1} = X^n$

![](https://github.com/freyakniglty/algorithm/blob/master/images/2.png)

![](https://github.com/freyakniglty/algorithm/blob/master/images/4.png)



**解线性方程：**

$A = I-\lambda dtL$ 是一个稀疏矩阵。

可以试着用PBCG（preconditioned bi-conjugate gradient）来求解。



**滤波器的改进：**

可以考虑把$X^{n+1} = (I+\lambda dt L)X^n$  中的L 换成 $L^2,L^3,L^4$

![](https://github.com/freyakniglty/algorithm/blob/master/images/3.png)



### 自动抗收缩修复

如果只是扩散则会产生收缩，提出一种线性的L 和 $L\circ L$ 的算子去增强低频率，达到平衡这种收缩的目的。

**计算容积：**

$V = \frac{1}{6}\sum^{nbFaces}_{k=1} g_k.N_k$

其中 $g = (x^1_k+x^2_k,x_k^3)/3$ , $N_k = \vec x^1_kx^2_k\land \vec x^1_kx_k^3$



**保持容积：**

将第一步得到的新的模型 volume $V^n$ 上所有顶点位置乘以 $\beta = (V^0/V^n)^{1/3}$

其中$V^0$ 是原始容积。

这样就可以将容积保证在初始值。

其作用在于，当第一步光滑完成后，增强原来低频率的地方来补偿削弱的高频率的地方。

也可以在特殊需要下，自行选择这个不变量。



                                                                                     

### 精确的扩散过程，改进前面的方法

**前面的不足：**

前面用到的umbrella operator （伞算子），对于高频率的和低平率有相同的拉普拉斯近似。如下图：

![](https://github.com/freyakniglty/algorithm/blob/master/images/5.png)



如果使用显式积分，则由以下不等式限制。

时间步长取决于 :

$dt \le \frac{min(|e|)^2}{2\lambda}$

其中 min(|e|)是最小边的长度



**改进方法：对1D的热力方程的模拟**

**1D 热力方程：** $x_t = x_{uu}$

![](https://github.com/freyakniglty/algorithm/blob/master/images/6.png)

上图指出，如果在边界上没有妥当地处理，会产生噪声造成非常大的误差。

如果我们对二阶导数求近似，就可以得到比常数的情况下更准确的结果。

extension of finite difference：(有限差分)

$ (x_{uu})_{i} = \frac{2}{\delta +\Delta}(\frac{x_{i-1}-x_i}{\delta}+\frac{x_{i+1}-x_i}{\Delta})$

当$\delta = \Delta$ 则得到usual finite difference（一般的有限差分）



**扩展到3D的情况：**

与1D热力方程相关的拉普拉斯方程近似：Fujiwara[fuj95] 提出这个公式

$L(x_i) = \frac{2}{E}\sum_{j\in N_1(i)} \frac{x_j-x_i}{e_{ij}}$   （11）

其中$E = \sum_{j\in N_1(i)}|e_{ij}| $ ， $ |e_{ij}|$ 是 边 $e_{ij}$ 的长度。

把（11）叫做 **依赖测度的伞算子**（scale-dependent umbrella operator）

但是这个算子不再是线性的了，但是在光滑的过程中，边的长度并不会有太大的改变。我们就可以近似地得到系数矩阵 $A = (I-\lambda dtL)$ 保持不变在一个积分步骤内。

![](https://github.com/freyakniglty/algorithm/blob/master/images/7.png)



积分次数（步骤所需次数）依赖于最长和最短边长的比率。



### 曲率流减少噪声

在差分方程中，扩散是与曲率流非常相关的。



**曲率流：**

我们倾向于不依赖参数化，只用到曲面的本质特性。

曲率流在曲面法向量上移动用一个平均曲率，来光滑曲面：

$\frac{\partial x_i}{\partial t} = - \bar{\kappa_i} n_i$

这个平均曲率也可以替换成别的，但一般用平均曲率。

注意不要在平坦的地方使用这个，因为这个时候曲率为0。



**对曲率流的网格上的近似：**

曲率法向 $\bar\kappa n$的定义

$ \frac{\nabla A}{2A} = \bar{\kappa} n $

这里 A是一个在顶点的小区域的面积。

![](https://github.com/freyakniglty/algorithm/blob/master/images/8.png)

得到一个离散的表达通过微分：

$-\bar\kappa n = \frac{1}{4A}\sum_{j\in N_1(i)}(\cot \alpha_j + \cot\beta_j)(x_j-x_i)$

如下图：

![](https://github.com/freyakniglty/algorithm/blob/master/images/9.png)

其中A是以$x_i$ 为公共顶点的区域的面积之和。



**对边界的处理：**

对于有洞没有闭合的网格，我们分开处理。一个方法是用（11）来直接处理，因为平均曲率对于这样的顶点是没有意义的。第二个方法是可以设置虚拟顶点。



**可以对 $\bar\kappa n$ 进行归一化处理：**

$(\bar\kappa n)_{normalized} = \frac{1}{\sum_j(\cot \alpha^l_j+\cot \alpha^r_j)} \sum_j(\cot \alpha^l_j+\cot \alpha^r_j)(X_i - X_j) $

**结果图对比：**

![](https://github.com/freyakniglty/algorithm/blob/master/images/10.png)



