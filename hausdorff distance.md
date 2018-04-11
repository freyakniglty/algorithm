**用途**：测量和更新简化的模型和原始模型之间的hausdorffDistance,提供精确的误差控制。

**前人工作**：

- Rossignac 和 Borrel 提出 vertex clustering。
- Cohen 的 simplification envelopes 。Zelinka and Garland 使用permission grids修改了这个方法。
- Popovic and hoppe ，Garland and Heckbert提出 vertex pair contraction 。成了非常普遍的方法。
- Klein 第一次使用Hausdorff distance。Borodin 结合了generalized pair contractions (vertex pair contraction) 得到了更高质量的结果。
- Cignoni 提出比较简化方法好坏的方法。Aspert 提出提速方法。

**术语**：

$d(p,S^`)=\min_{p^`\in S^`}d(p,p^`)$

这里 $d(p,p^`)$ 是欧氏距离

**几何距离**，也叫 ***one-sided ,single-sided Hausdorff distance*** 在两个曲面之间被定义为：

$d(S,S^`)=\max_{p\in S}d(p,S^`)$

注意，这个距离并不是对称的 即 $d(S,S^`) \ne d(S^`,S)$

***symmetrical* *Hausdorff distance*** 被定义为：

$，d_s(S,S^`) =max(d(S,S^`) ， d(S^`,S))$





**主要思想：**调整模型密度，用于计算实际在对应区域的几何偏差距离。

**主要目标：**只在最大距离在两个对象之间符合期望的地方画样本（samples）。



**数据处理：**

为了快速得到有大的几何距离的区域，我们将在 两个voxel grids（体素网格？）上检索三角形网格。

网格(grid)的规格依赖于bounding boxes 和三角形的个数。我们平均让十个三角形组成一个单元。

得到分辨率 **$r = \sqrt{\frac { \# triangles}{10*6} }$**

为了给找到大的距离的体元加速，可以用八叉树（octree）结构



**主要算法：**

开始，我们设定Hausdorff distance 为 0.

开始在同时在两个网格的八叉树结构中遍历，计算每一个单元与其他所有相同级别的单元的距离，为了找到到另一个网格上最近的一个。

如果对现在的一个单元，最近的另外一个单元被找到，我们可以计算在这些点之间距离的最大和最小值。

如果最小值比现在的Hausdorff 距离大，就更新它为Hausdorff距离。

如果最大值比现在的Hausdorff距离小或者相等，就略过它。

八叉树使用优先级，先考虑包含比较大的距离的三角形的单元。

当遍历到叶单元，我们选取所有包含的三角形，根据它们的几何距离的最大值，把它们插入到优先队列（priority queue）。根据它们的最小距离我们再次更新hausdorff distance.

为了避免相同三角形的重复插入，我们标记三角形然后只处理未标记的。当这个队列变空了或者剩下的三角形和单元的最大距离都小于已经找到的hausdorff距离，遍历结束。

以下是**伪代码**：

```c++
inError=0

AddToQueue(RootCellA)

AddToQueue(RootCellB)

while(QueueNotEmpty)

     GetCellWithHighestMaxDistance

     UpdateMinError

if(LeafNode)

     InsertTrianglesIntoQueue

else

    InsertChildrenIntoQueue

return minError

```



**定义距离：**

1.**Cell_Based Distance (单元距离)**

为了快速找到最近的单元，当遍历八叉树从它的结点到它的子结点时，我们记录符合以下条件的单元：最小距离小于对最近的单元的最大距离。然后我们只需要检查这些结点，当要计算所有单元的子结点的距离。

为了简化距离计算，我们使用单元bounding box去构建网格。立方体的网格可以进一步简化计算。

2.**三角形的距离**

为了计算一个三角形到另一个mesh的几何距离的上界和下界，我们第一步需要计算它的顶点的距离。

如果一个顶点已经在正处理的体网格单元内部，我们可以使用记录的最近单元去找到合适的三角形。

如果它在正在处理的体网格单元之外，我们降级去找到离这个顶点最近的单元。

然后我们从被包含在最近的单元的三角形开始计算三角形之间的距离。当最近的点的距离被找到，它的距离小于所有剩下的单元，正在处理的顶点的距离就被找到了。同样记录已处理和未处理的三角形以避免重复的计算。

在三角形的三个顶点被计算完毕之后，三角形的最小几何距离就是$||V_i - P_i||$ 的最大值。三角形的最大几何距离是 顶点距离和B到 $P_i$ 的距离 的最大值。

![]()

得到：

$d \ge \max^3_{i=1}(||V_i - P_i||) = d_{min}$

$d \le \max(H_{min},\max^3_{i=1}(||B-P_i||))$

还有：

$ d \le \min^3_{i=1}(\max^3_{j=1}(||V_i - P_j||))$

如果三个顶点的最近点落在同一个三角形上，最大顶点距离就是现在的三角形的几何距离。不然三角形就放入priority queue，（它的顶点落在不同的三角形上）。当它被处理时，被分成几块来处理。为了避免重复计算，将分好的子三角形放入priority queue而不是原三角形。

![]()

```c++
CalculateSubdivisionBasePoints
for(allChildTriangles)
   minDistance=max(vertexDistances)
   if(AllBasePointsOnSameTriangle)
      maxDistance=minDistance
   else
      maxDistance=max(barycenterDistances)
   InsertIntoQueue
```



流程：

控制Haudorff error.

只有影响到现在操作的网格的部分才需要被考虑。因此受到影响的简化模型的三角形才被插入队列，然后只考虑与原模型共同三角形的bounding box。因为原模型相邻的三角形可能受到影响，所以就扩大boundingbox的范围。

没有必要计算精确的几何距离误差，只要设立阈值，只考虑修正超过阈值的部分即可。

![]()

先计算quadric error （二次误差）。

新顶点到简化的模型的距离误差超出hausdorff距离的两倍，就拒绝。然后再计算新顶点到原始模型的距离误差，如果超出一个特定的阈值，则拒绝。如下图。

![]()
