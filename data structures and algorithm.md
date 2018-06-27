## 数据结构与算法

### 基本概念
#### 什么是数据结构
- 数据结构
1.数据对象在计算机中的组织方式：逻辑结构、物理存储结构
2.数据对象必定与一系列加在其上的操作相关联
3.完成这些操作所用的方法就是算法
- 抽象数据类型
1.数据类型：数据对象集、数据集合相关联的操作集
2.抽象：描述数据类型的方法不依赖于其具体实现
即只描述数据对象集合相关操作集‘是什么’，并不涉及‘如何做到’的问题
#### 什么是算法
- 算法定义
1.一个有限指令集
2.接受一些输入（有些情况下不需要输入）
3.产生输出
4.一定在有限步骤之后终止
5.每一条指令必须有充分明确的目标，在计算机能处理范围之内且描述不能依赖于任何一种计算机语言以及具体的实现
- 好的算法
1.空间复杂度S(n)：根据算法写成的程序在执行时占用存储单元的长度
2.时间复杂度T(n)：根据算法写成的程序在执行时耗费时间的长度
- 复杂度
1.T1(n)+T2(n)=max(O(f1(n)),O(f2(n)))
2.T1(n)×T2(n)=O(f1(n)×f2(n))
3.for循环的复杂度等于循环次数乘以循环体复杂度
4.if语句复杂度取决于if的条件判断复杂度和两个分支部分的复杂度，取三者最大的复杂度
#### 应用实例：最大子列和
- 算法1：
```python
def MaxSubseqSum1(a,N):
  #a是给定整数序列，N是序列长度
  MaxSum=0
  for i in range(N):
    for j in range(i,N):
      ThisSum=0
      for k in range(i,j+1):
        ThisSum+=a[k]
      if ThisSum>MaxSum:
        MaxSum=ThisSum
  return MaxSum
```
复杂度T(N)=O(N^3)
- 算法2
```python
def MaxSubseqSum2(a,N):
  MaxSum=0
  for i in range(N):
    ThisSum=0
    for j in range(i,N):
      ThisSum+=a[j]
      if ThisSum>MaxSum:
        MaxSum=ThisSum
  return MaxSum
```
复杂度T(N)=O(N^2)
- 算法3 MapReduce
迭代拆分成两部分，算出左半部分最大子列和，右半部分最大子列和和交叉部分最大子列和，取最大值
复杂度T(N)=O(NlogN)
- 算法4 在线处理
```python
def MaxSubseqSum3(a,N):
  MaxSum=0
  ThisSum=0
  for i in range(N):
    ThisSum+=a[i]#向右累加
    if ThisSum>MaxSum:
      MaxSum=ThisSum#发现更大和更新
    elif ThisSum<0:
      continue#子列和为负则抛弃
  return MaxSum    
```
复杂度T(N)=O(N)
