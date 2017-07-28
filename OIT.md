
# OIT

在游戏场景中，如果存在透明物体，怎么绘制正确的半透明效果是个难题。以前的解决方案通常是针对透明物体进行深度排序，然后从后往前渲染输出到color buffer中，但是这种基于模型的排序两个问题：

* 很难排序：如果存在物体交叉的情况，怎么保证哪个物体是在前或者后面呢？
* 不可能排序: 多个三角形交叉的情况，只能通过排序片元的方式


发展过程中，有人提出了depth peeling的方式进行顺序无关透明方式，但是这个方法带来的问题是buffer的开销比较大，设定多少层peeling也是取决于效果的好坏。而且效率上也较差，所以
有人提出了基于链表的形式进行OIT。

使用逐像素链表的OIT的思想是用两个buffer，start offset buffer保存头指针，linkedlist buffer保存半透明物体的片元信息（颜色，深度，next指针），然后排序当前片元的链表，从后往前混合，最终输出混合效果。

### 算法过程

* 正确输出不透明物体的光照结果到颜色buffer中；

* 创建链表：
start offset buffer可以创建为一通道的整数纹理，使用原子计数器操作为每个像素分配头指针Id,atomicCounterIncrement和imageAtomicExchange，
而linkedlist buffer则保存当前半透明物体的片元信息；

* 遍历链表：
输入上一步中的两个buffer，然后根据start offset buffer的头指针id取链表buffer中的信息，如果next有值，继续后移链表操作。
把当前像素的所有片元都保存在一个临时数组中，然后进行一个简单排序方式。最后进行深度值从后往前混合，并注意背景颜色加上混合。
得到最终的结果就是正确的顺序无关透明结果。

---

 [Order-Independent Transparency in DX 11](https://graphics.stanford.edu/wikis/cs448s-10/FrontPage?action=AttachFile&do=get&target=CS448s-10-11-oit.pdf)