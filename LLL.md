# LLL

* 主要解决什么问题

延迟光照中不能处理透明物体或者粒子特效，如烟雾的效果，因为它依靠深度buffer信息，如果透明物体更改深度buffer缓冲区，那么遮挡的不透明物体因为深度测试则不会显示。 以前的做法是对透明物体不进行光照处理或者执行一个额外的前向pass来渲染透明物体，不过这也带来的额外的性能开销。

* 怎么解决这个问题？

针对每个光源建立链表，光栅化的片元遍历当前像素的链表，如果在光源几何体中，执行光照算法。 因为LLL算法仅计算被光源影响的像素，从而提升性能的效率。而且，针对没被编码进深度缓存的透明几何体，也能有相同的访问机制。Light Linked List是GPU生成和访问，对CPU的要求很低。

### 光源链表结构体定义

```c++
struct SLightLinkedList
{
	float m_MinDepth;
	float m_MaxDepth;
	uint m_LightIndex;
	uint m_Next;
}
```

优化光源链表，可以通过**f32tof15**和**floatBitsToUint**把深度信息保存在uint的高16bit和低16bit。而光源的index使用高8位，实际场景中，几乎没有一个像素超过72个光源的，所以这里够用了。剩下的next就用来保存光源指针。

```c++
struct SLightLinkedList
{
	uint m_DepthInfo;
	uint m_IndexNext;
}
```

### 算法步骤

* 创建光源链表(Light Shell, Soft depth test, depth bound)

透明物体场景中存在三种情况，一个是在不透明物体背后，一个是物体前面，一个是与物体相交。 如果执行硬件深度测试的话，那么在不透明物体背后的片元都不能写入buffer中。因为深度测试失败，所以需要执行软件深度测试。这里使用两边pass来获取光源的最小深度值和最大深度值。 第一遍pass保存光源前向的深度值，打开背面culling，使用**gl_FrontFacing**判断当前是否是正面片元。 同理在第二遍pass打开前向culling保存前向深度值。注意的是，这里需判断两个片元是否来自同一光源index。

* 遍历光源链表

不光是针对透明或不透明物体，都可以访问光源链表。首先转换视口的的坐标为LLL的分辨率坐标，因为两者分辨率是不一致的。 然后取出light start offset 缓存的头指针，然后判断当前片元的深度值，是否在光源求的mindepth和maxdepth之间，如果在就访问在SSBO中光源信息进行光照。 如果不在则不进行处理，这不光解决传统模板缓存不能处理透明效果的问题，执行效率也比他快速。


* 优化

减少分辨率：如果针对1920*1080的分辨率，一个片元如果平均有32个光源影响，那么缓存空间 **1920 × 1080 × 32 × LightFragmentLink** = 506.25MB。实际上，可以生成LLL为1/4或者1/8的分辨率，这样能大幅度降低内存开销。

深度缓存的下采样：因为需要匹配LLL的分辨率，所以下采样depth buffer到1/8的分辨率，使用max避免light信息丢失，因为z-culling。还有使用**GatherRed**一次取4个纹理值。

### 结论

LLL动态简化光照的pipeline，而且允许我们对于透明物体和粒子特效实施光照处理。使用LLL，可以改善延迟渲染的性能和减少内存开销。除此之外，LLL的灵活性允许应用光照到皮肤，毛发等材质上。