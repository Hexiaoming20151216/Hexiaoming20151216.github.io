<script type="text/javascript" async
  src="https://cdn.mathjax.org/mathjax/latest/MathJax.js?config=TeX-MML-AM_CHTML">
</script>
# 图像实时赋色与量测更新

# 简介

前端里程计中引入了图像数据，主要用于实时赋色功能和基于图像信息的量测更新。现已适配手持设备LigripOne和LigripMid360，相机模型实现KB模型和MEI模型。

本文将代码中适用的理论公式做一个记录，方便后期优化更新代码时查阅参考之用。

# 实时赋色

实时赋色流程简要介绍：图像位姿来自于上一帧激光位姿加上IMU积分所得的预测位姿，当前帧激光点转换到最近的相机坐标系下，再相机坐标系下的激光点重投影到图像平面，获取该像素的rgb信息，完成激光点的赋色。

## MEI相机模型

[请至钉钉文档查看附件《single\_viewpoint\_calib\_mei\_07.pdf》](https://alidocs.dingtalk.com/i/nodes/lyQod3RxJK3P9wKpsMqrN0yXJkb4Mw9r?doc_type=wiki_doc&iframeQuery=anchorId%3DX02lrk9xi82rgt49qae52)

[VinsFusion中的MEI模型解析 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/414047132)

[相机内参模型Mei/omni-directional详解 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/532505565)

![image.png](https://alidocs.oss-cn-zhangjiakou.aliyuncs.com/res/WgZOZwWGYrMLlLX8/img/c0dfdf22-40f2-44a9-971b-af0c7c8a6674.png)

![image.png](https://alidocs.oss-cn-zhangjiakou.aliyuncs.com/res/WgZOZwWGYrMLlLX8/img/9398ac2b-e839-4c5a-b64a-375c9af948b4.png)

### 空间三维点到像素平面的投影

相机坐标系（“右下前”）下的一个点$P\_{cam}$（上图中的P），投影到像素平面的步骤如下：

相机坐标系下三维点坐标为：

$P\_{cam} = (x,y,z)^T$

对$P\_{cam}$进行归一化处理，即将点映射到上图中的归一化球面上，即为上图中的$P\_{s}$，坐标原点为$C\_{m}$:

$r=\sqrt{x^2+y^2+z^2}$

$P\_{s} = (x/r, y/r ,z/r )^T$

对坐标原点在Z轴添加一个偏移$\xi$（这里对应镜像参数，代码中的xi\_）， 使得坐标原点由$C\_{m}$变为$C\_{p}$，那么归一化球面上的三维点$P\_{s}$，在以$C\_{p}$为原点的新坐标变为：

$P\_{s} = (x/r, y/r ,z/r+\xi )^T$

对$P\_{s}$新坐标的Z坐标归一化，此时的归一化平面为上图中的$\pi\_{mu}$, $P\_{s}$投影到$\pi\_{mu}$平面的点坐标为：

$m\_{u} = (x / r(z/r+\xi ), y/r(z/r+\xi ) ,1 )^T= (x / (z+r\xi ), y/(z+r\xi ) ,1 )^T$

剩下的就是普通的针孔相机模型了，加上径向畸变和切向畸变，去畸变公式如下：

$m\_{u} = (x\_{mu},y\_{mu} ,1 )^T$

$r\_{mu}=\sqrt{(x\_{mu}^2+y\_{mu}^2)}$

$\varDelta x\_{mu} = x\_{mu} +k\_1r\_{mu}^2+k\_2r\_{mu}^4+k\_3r\_{mu}^6+2p\_1x\_{mu}y\_{mu}+p\_2(r\_{mu}^2+2x\_{mu}^2)$

$\varDelta y\_{mu} = y\_{mu} +k\_1r\_{mu}^2+k\_2r\_{mu}^4+k\_3r\_{mu}^6+2p\_2x\_{mu}y\_{mu}+p\_1(r\_{mu}^2+2y\_{mu}^2)$

$m\_{d} = (x / (z+r\xi )+\varDelta x\_{mu}, y/(z+r\xi )\varDelta y\_{mu} ,1 )^T$

最后，将去畸变之后的点通过针孔相机模型投影到像素平面：

$p\_{uv}=Km\_d= \begin{bmatrix}    f\_x & 0 & c\_x \\    0 & f\_y & c\_y \\    0 & 0 &1 \end{bmatrix} m\_d$

这部分代码实现如下：

![image.png](https://alidocs.oss-cn-zhangjiakou.aliyuncs.com/res/WgZOZwWGYrMLlLX8/img/33c9ad97-0531-40d6-88c0-8b8c14195dcc.png)

![image.png](https://alidocs.oss-cn-zhangjiakou.aliyuncs.com/res/WgZOZwWGYrMLlLX8/img/ec6d4753-1b34-4952-8fc2-f2990138a50d.png)

### 像素平面到空间三维点的反投影

已知像素平面上的一个点$p\_{uv}$，求坐标原点为$C\_{m}$（真实物理相机坐标中心）的归一化球面坐标系（未偏移）下的坐标$P\_{s}$

这部分代码实现如下：

![image.png](https://alidocs.oss-cn-zhangjiakou.aliyuncs.com/res/WgZOZwWGYrMLlLX8/img/03f91124-ac99-4716-b403-6a06ce70a67e.png)

## KannalaBrandt8相机模型

[ORBSLAM3(六) Kannala\_Brandt鱼眼相机模型\_orb-slam 相机模型-CSDN博客](https://blog.csdn.net/AsJHJYHJYG/article/details/123332431)

[KannalaBrandt8鱼眼相机模型-CSDN博客](https://blog.csdn.net/xhtchina/article/details/126115380)

[一文详解分析鱼眼相机投影成像模型和畸变模型 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/511284263)

![image.png](https://alidocs.oss-cn-zhangjiakou.aliyuncs.com/res/WgZOZwWGYrMLlLX8/img/73185193-46d8-443a-af4a-03ed9c2a7332.png)

### 空间三维点到像素平面的投影

相机坐标系（“右下前”）下的一个点$P\_{cam}$，投影到像素平面的步骤如下：

相机坐标系下三维点坐标为：

$P\_{cam} = (x,y,z)^T$

求经纬度：

$\theta = atan({{\sqrt{x^2+y^2}} \over z})$

$\phi = atan({y \over x})$

去畸变系数r:

$r = \theta+k\_1\theta^3+k\_2\theta^5+k\_3\theta^7+k\_4\theta^9$

像素坐标：

$u=f\_x\*r\*cos(\phi)+c\_x$

$v=f\_y\*r\*sin(\phi)+c\_y$

这部分代码实现如下：

![image.png](https://alidocs.oss-cn-zhangjiakou.aliyuncs.com/res/WgZOZwWGYrMLlLX8/img/a082832c-66de-4cd9-86da-9e5fecf01fa6.png)

### 像素平面到空间三维点的反投影

已知像素平面上的一个点$p\_{uv}$，求归一化球面坐标系下的坐标$P\_{s}$

像素平面上的一个点$p\_{uv}$坐标为：

$p\_{uv}=(u,v)$

归一化平面坐标:

$$
p_d = (x, y) = \left( \frac{u - c_x}{f_x}, \frac{v - c_y}{f_y} \right)
$$

\( p_d = (x, y) = \left( \frac{u - c_x}{f_x}, \frac{v - c_y}{f_y} \right) \)



可以求得带畸变的入射角：

$\theta\_d = {\sqrt{x^2+y^2}}$

$\phi = atan(y, x)$

去畸变后得到$\theta$

最后得到单位球面坐标：

$x = sin(\theta)cos(\phi)$

$y = sin(\theta)cos(phi)$

$z = cos(\theta)$

这部分代码实现如下：

![image.png](https://alidocs.oss-cn-zhangjiakou.aliyuncs.com/res/WgZOZwWGYrMLlLX8/img/6e1b8a26-3462-4c43-b56f-c9a6680588fa.png)

# 基于图像观测的量测更新

[四元数扰动与求导：左扰动&右扰动 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/603214072)

基本流程思路：利用点云的局部地图，获取特征点在世界坐标系下的三维坐标，光流跟踪，得到具备三维信息的特征点在当前帧的像素坐标，残差定义：1. MEI模型为三维点在当前帧的单位球面上的距离；2. KB 模型为三维点在当前帧的重投影误差。

## MEI模型量测更新

todo: 补充理论公式推导

reference: [VINS-MONO ProjectionFactor代码分析及公式推导 - 念念无滞 - 博客园 (cnblogs.com)](https://www.cnblogs.com/glxin/p/11990551.html)

### MEI模型相机观测残差定义：

现在LAM算法中MEI模型的相机观测残差定义同VINS中的定义一样，即三维目标点和像素跟踪点在单位球面上的切向距离。具体推导流程如下所述：

我们以一个点为例：一个在世界坐标系下的点$P\_{w}$（这里的世界系坐标来自于激光点云），在上一帧图像中被观测到的像素坐标为$UV\_{last}$，在当前帧图像中被观测到的像素坐标为$UV\_{cur}$。

点$P\_{w}$转换到当前图像时刻的IMU坐标系下的坐标为：

$P\_{imu}=(R^w\_i)^{-1}(P\_{w}-t^w\_i)$

点$P\_{imu}$转换到当前图像时刻的相机坐标系下的坐标为：

$P\_{cam}=R^c\_iP\_{imu}+t^c\_i$

将相机系下点$P\_{cam}$转换到相机以相机光心为原点的球面，的归一化坐标点

$dist = P\_{cam}.norm()= {\sqrt{x\_{c}^2+y\_{c}^2+z\_{c}^2}}$

$p\_{norm}=({{x\_{c}}\over dist}, {{y\_{c}}\over dist}, {{z\_{c}}\over dist}) =({{x\_{c}}\over  {\sqrt{x\_{c}^2+y\_{c}^2+z\_{c}^2}}}, {{y\_{c}}\over  {\sqrt{x\_{c}^2+y\_{c}^2+z\_{c}^2}}}, {{z\_{c}}\over  {\sqrt{x\_{c}^2+y\_{c}^2+z\_{c}^2}}})$

另一边，根据在当前帧图像中被观测到的像素坐标$UV\_{cur}$，以及上一节中MEI模型像素坐标到单位球面的反投影和去畸变公式可得$p\_{sphere}$  , 这里像素坐标定了之后，其在球面上的坐标亦为定值。

**单位平面正切平面正交基：**

图中$b\_{1}$ ，$b\_{2}$ 为以相机光心为原点的单位球面上正切平面标准正交基，单位球面上一点$\=P\_{l}^{cj}$为图像像素点投影到单位球面上的估计值，在优化增量$r\_{c}$ 的作用下，点位置更新到$P\_{l}^{cj}$

![image.png](https://alidocs.oss-cn-zhangjiakou.aliyuncs.com/res/WgZOZwWGYrMLlLX8/img/0f8ce587-929d-4dc8-a77c-3630361f69e6.png)

$b\_{1}$ ，$b\_{2}$ 这组正交基可以通过球半径和向量$\=P\_{l}^{cj}$点乘（内积投影），叉乘（外积垂直）得到：

![image.png](https://alidocs.oss-cn-zhangjiakou.aliyuncs.com/res/WgZOZwWGYrMLlLX8/img/fa351d85-e490-4eef-95e8-187b91c16385.png)对应代码为：todo: 修改正交基为重投影三维点（即通过待优化状态量转换得到的单位球面投影点）处的标准正交基

![image.png](https://alidocs.oss-cn-zhangjiakou.aliyuncs.com/res/WgZOZwWGYrMLlLX8/img/3157e24c-3e2c-4b6d-afd6-b27cda0914ad.png)

残差定义为：单位球面上的观测点和目标点之间的差值在$b\_{1}$ ，$b\_{2}$ 这组正交基上的投影。

### 雅可比推导：

![image.png](https://alidocs.oss-cn-zhangjiakou.aliyuncs.com/res/WgZOZwWGYrMLlLX8/img/96985174-9245-40de-a210-70c65bf61101.png)

残差计算和雅可比计算的代码实现如下：

![image.png](https://alidocs.oss-cn-zhangjiakou.aliyuncs.com/res/WgZOZwWGYrMLlLX8/img/e6b8a3bc-8143-4ef8-9681-e6ae9acef4f6.png)

![image.png](https://alidocs.oss-cn-zhangjiakou.aliyuncs.com/res/WgZOZwWGYrMLlLX8/img/03b43e40-f4e6-4fd5-a825-77203cd6d54f.png)

![image.png](https://alidocs.oss-cn-zhangjiakou.aliyuncs.com/res/WgZOZwWGYrMLlLX8/img/2ebacead-9d29-47a0-95e6-e5abadb3b5ea.png)

## KB模型量测更新

这里我们需要优化的状态量主要为当前图像时刻的$R\_{imu2world}$, $t\_{imu2world}$, 观测为上一帧中的三维点在当前帧被光流跟踪得到的像素坐标$p\_{uv}$。

我们单独分析一个点$p\_{w}$, 它在世界系下的坐标为：

$p\_{w}=(x\_w,y\_w,z\_w)$

转换到当前图像时刻IMU坐标系下的坐标为：

$p\_{imu}=R\_{imu2world}^{-1}(p\_{w}-t\_{imu2world})$

利用相机和IMU的外参可得，该点在相机坐标系下的三维坐标为：

$p\_{cam}=R\_{imu2cam}p\_{imu}+t\_{imu2cam} = R\_{imu2cam}R\_{imu2world}^{-1}(p\_{w}-t\_{imu2world})+t\_{imu2cam}$

将$p\_{cam}$投影到像素平面：

$p\_{uv}=project(p\_{cam})=project(R\_{imu2cam}R\_{imu2world}^{-1}(p\_{w}-t\_{imu2world})+t\_{imu2cam})$

从而残差项为：

$residual\_{uv}=project(R\_{imu2cam}R\_{imu2world}^{-1}(p\_{w}-t\_{imu2world})+t\_{imu2cam}) - cur\_{uv}$

雅可比求解：

对旋转求导：

$\tfrac{\partial (residual\_{uv})}{\partial R\_{imu2world}} =\tfrac{\partial (project(R\_{imu2cam}R\_{imu2world}^{-1}(p\_{w}-t\_{imu2world})+t\_{imu2cam}) - cur\_{uv})}{\partial R\_{imu2world}}\\ = \tfrac{\partial (project(p\_{cam}))}{\partial R\_{imu2world}} = \tfrac{\partial (project(p\_{cam}))}{\partial p\_{cam}} \* \tfrac{\partial (p\_{cam})}{\partial R\_{imu2world}}\\= \tfrac{\partial (project(p\_{cam}))}{\partial p\_{cam}} \*R\_{imu2cam}\* \tfrac{\partial (R\_{imu2world}^{-1}(p\_{w}-t\_{imu2world})}{\partial R\_{imu2world}}$

已知：$\tfrac{\partial R^T\*P}{\partial \delta\theta} =\lfloor R^T\*P\rfloor$，所以：

$\tfrac{\partial (residual\_{uv})}{\partial R\_{imu2world}} = \tfrac{\partial (project(p\_{cam}))}{\partial p\_{cam}} \*R\_{imu2cam}\* (  (R\_{imu2world}^{-1}(p\_{w}-t\_{imu2world}))^T\\ = \tfrac{\partial (project(p\_{cam}))}{\partial p\_{cam}} \*R\_{imu2cam}\* \lfloor  p\_{imu}\rfloor$

其中， $\tfrac{\partial (project(p\_{cam}))}{\partial p\_{cam}}$（对应公式不想推了，直接看代码吧），对应代码如下：

![image.png](https://alidocs.oss-cn-zhangjiakou.aliyuncs.com/res/WgZOZwWGYrMLlLX8/img/bd3d9c66-f2ae-4af0-ac98-69d95e59e579.png)

对平移求导：

$\tfrac{\partial (residual\_{uv})}{\partial t\_{imu2world}} =\tfrac{\partial (project(R\_{imu2cam}R\_{imu2world}^{-1}(p\_{w}-t\_{imu2world})+t\_{imu2cam}) - cur\_{uv})}{\partial t\_{imu2world}}\\ = \tfrac{\partial (project(p\_{cam}))}{\partial p\_{cam}} \* \tfrac{\partial (p\_{cam})}{\partial t\_{imu2world}}\\= -\tfrac{\partial (project(p\_{cam}))}{\partial p\_{cam}} \*R\_{imu2cam}\* R\_{imu2world}^{-1}$

此部分对应代码如下：

![image.png](https://alidocs.oss-cn-zhangjiakou.aliyuncs.com/res/WgZOZwWGYrMLlLX8/img/d64c1901-aa09-4490-82f2-66f4c4b94431.png)

效果：

![image.png](https://alidocs.oss-cn-zhangjiakou.aliyuncs.com/res/WgZOZwWGYrMLlLX8/img/83f4f7d3-4880-4d41-998f-907ba003fd9e.png)