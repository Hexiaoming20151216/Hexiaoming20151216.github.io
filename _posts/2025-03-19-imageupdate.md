<script type="text/javascript" async
  src="https://cdn.mathjax.org/mathjax/latest/MathJax.js?config=TeX-MML-AM_CHTML">
</script>
# 图像实时赋色与量测更新

# 简介

前端里程计中引入了图像数据，主要用于实时赋色功能和基于图像信息的量测更新。相机模型实现KB模型和MEI模型。

## MEI相机模型

[《single_viewpoint_calib_mei_07.pdf》](assets/files/single_viewpoint_calib_mei_07.pdf)

[VinsFusion中的MEI模型解析 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/414047132)

[相机内参模型Mei/omni-directional详解 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/532505565)

![image.png](https://alidocs.oss-cn-zhangjiakou.aliyuncs.com/res/WgZOZwWGYrMLlLX8/img/c0dfdf22-40f2-44a9-971b-af0c7c8a6674.png)

![image.png](https://alidocs.oss-cn-zhangjiakou.aliyuncs.com/res/WgZOZwWGYrMLlLX8/img/9398ac2b-e839-4c5a-b64a-375c9af948b4.png)

### 空间三维点到像素平面的投影

相机坐标系（“右下前”）下的一个点$$P_{cam}$$上图中的P，投影到像素平面的步骤如下：

相机坐标系下三维点坐标为：

$$p_{cam} = (x,y,z)^T$$

对$$p_{cam}$$进行归一化处理，即将点映射到上图中的归一化球面上，即为上图中的$$P_{s}$$，坐标原点为$$C\_{m}$$:

$$r=\sqrt{x^2+y^2+z^2}$$

$$P_{s} = (x/r, y/r ,z/r )^T$$

对坐标原点在Z轴添加一个偏移$$\xi$$（这里对应镜像参数，代码中的xi\_）， 使得坐标原点由$$C\_{m}$$变为$$C\_{p}$$，那么归一化球面上的三维点$$P_{s}$$，在以$$C\_{p}$$为原点的新坐标变为：

$$P_{s} = (x/r, y/r ,z/r+\xi )^T$$

对$$P_{s}$$新坐标的Z坐标归一化，此时的归一化平面为上图中的$$\pi\_{mu}$$, $$P_{s}$$投影到$$\pi\_{mu}$$平面的点坐标为：

$$m_{u} = (x / r(z/r+\xi ), y/r(z/r+\xi ) ,1 )^T= (x / (z+r\xi ), y/(z+r\xi ) ,1 )^T$$

剩下的就是普通的针孔相机模型了，加上径向畸变和切向畸变，去畸变公式如下：

$$m_{u} = (x_{mu},y_{mu} ,1 )^T$$

$$r_{mu}=\sqrt{(x_{mu}^2+y_{mu}^2)}$$

$$\varDelta x_{mu} = x_{mu} +k_1r_{mu}^2+k_2r_{mu}^4+k_3r_{mu}^6+2p\_1x_{mu}y_{mu}+p\_2(r_{mu}^2+2x_{mu}^2)$$

$$\varDelta y_{mu} = y_{mu} +k_1r_{mu}^2+k_2r_{mu}^4+k_3r_{mu}^6+2p\_2x_{mu}y_{mu}+p\_1(r_{mu}^2+2y_{mu}^2)$$

$$m_{d} = (x / (z+r\xi )+\varDelta x_{mu}, y/(z+r\xi )\varDelta y_{mu} ,1 )^T$$

最后，将去畸变之后的点通过针孔相机模型投影到像素平面：

$$p_{uv}=Km_d= \begin{bmatrix}    f_x & 0 & c_x \\    0 & f_y & c_y \\    0 & 0 &1 \end{bmatrix} m_d$$

### 像素平面到空间三维点的反投影

已知像素平面上的一个点$$p_{uv}$$，求坐标原点为$$C\_{m}$$（真实物理相机坐标中心）的归一化球面坐标系（未偏移）下的坐标$$P_{s}$$


## KannalaBrandt8相机模型

[ORBSLAM3(六) Kannala\_Brandt鱼眼相机模型\_orb-slam 相机模型-CSDN博客](https://blog.csdn.net/AsJHJYHJYG/article/details/123332431)

[KannalaBrandt8鱼眼相机模型-CSDN博客](https://blog.csdn.net/xhtchina/article/details/126115380)

[一文详解分析鱼眼相机投影成像模型和畸变模型 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/511284263)

![image.png](https://alidocs.oss-cn-zhangjiakou.aliyuncs.com/res/WgZOZwWGYrMLlLX8/img/73185193-46d8-443a-af4a-03ed9c2a7332.png)

### 空间三维点到像素平面的投影

相机坐标系（“右下前”）下的一个点$$p_{cam}$$，投影到像素平面的步骤如下：

相机坐标系下三维点坐标为：
$$p_{cam} = (x,y,z)^T$$
求经纬度：
$$\theta = atan\left(\frac{\sqrt{x^2 + y^2}}{z}\right)$$
$$\phi = atan({y \over x})$$

去畸变系数r:

$$r = \theta+k_1\theta^3+k_2\theta^5+k_3\theta^7+k_4\theta^9$$

像素坐标：

$$u=f_x * r *cos(\phi)+c_x$$

$$v=f_y *r *sin(\phi)+c_y$$


### 像素平面到空间三维点的反投影

已知像素平面上的一个点$$p_{uv}$$，求归一化球面坐标系下的坐标$$P_{s}$$

像素平面上的一个点$$p_{uv}$$坐标为：

$$p_{uv}=(u,v)$$

归一化平面坐标:

$$
p_d = (x, y) = \left( \frac{u - c_x}{f_x}, \frac{v - c_y}{f_y} \right)
$$



可以求得带畸变的入射角：

$$\theta\_d = {\sqrt{x^2+y^2}}$$

$$\phi = atan(y, x)$$

去畸变后得到$$\theta$$

最后得到单位球面坐标：

$$x = sin(\theta)cos(\phi)$$

$$y = sin(\theta)cos(phi)$$

$$z = cos(\theta)$$


# 基于图像观测的量测更新

[四元数扰动与求导：左扰动&右扰动 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/603214072)

基本流程思路：利用点云的局部地图，获取特征点在世界坐标系下的三维坐标，光流跟踪，得到具备三维信息的特征点在当前帧的像素坐标，残差定义：1. MEI模型为三维点在当前帧的单位球面上的距离；2. KB 模型为三维点在当前帧的重投影误差。

## MEI模型量测更新

todo: 补充理论公式推导

reference: [VINS-MONO ProjectionFactor代码分析及公式推导 - 念念无滞 - 博客园 (cnblogs.com)](https://www.cnblogs.com/glxin/p/11990551.html)

### MEI模型相机观测残差定义：

MEI模型的相机观测残差定义同VINS中的定义一样，即三维目标点和像素跟踪点在单位球面上的切向距离。具体推导流程如下所述：

我们以一个点为例：一个在世界坐标系下的点$$p_{w}$$（这里的世界系坐标来自于激光点云），在上一帧图像中被观测到的像素坐标为$$UV_{last}$$，在当前帧图像中被观测到的像素坐标为$$UV_{cur}$$。

点$$p_{w}$$转换到当前图像时刻的IMU坐标系下的坐标为：

$$p_{imu}=R_{w2i}(p_{w}-t_{i2w})$$

点$$p_{imu}$$转换到当前图像时刻的相机坐标系下的坐标为：

$$p_{cam}=R_{i2c}p_{imu}+t_{i2c}$$

将相机系下点$$p_{cam}$$转换到相机以相机光心为原点的球面，的归一化坐标点

$$dist = p_{cam}.norm()= {\sqrt{x_{c}^2+y_{c}^2+z_{c}^2}}$$

$$
p_{norm} = \left( \frac{x_c}{dist}, \frac{y_c}{dist}, \frac{z_c}{dist} \right) =
\\
\left( \frac{x_c}{\sqrt{x_c^2 + y_c^2 + z_c^2}}, \frac{y_c}{\sqrt{x_c^2 + y_c^2 + z_c^2}}, \frac{z_c}{\sqrt{x_c^2 + y_c^2 + z_c^2}} \right)
$$

另一边，根据在当前帧图像中被观测到的像素坐标$$UV_{cur}$$，以及上一节中MEI模型像素坐标到单位球面的反投影和去畸变公式可得$$p\_{sphere}$$  , 这里像素坐标定了之后，其在球面上的坐标亦为定值。

**单位平面正切平面正交基：**

图中$$b\_{1}$$ ，$$b\_{2}$$ 为以相机光心为原点的单位球面上正切平面标准正交基，单位球面上一点$$\overline{P}_{l}^{cj}$$为图像像素点投影到单位球面上的估计值，在优化增量$$r\_{c}$$ 的作用下，点位置更新到$$P\_{l}^{cj}$$

![image.png](https://alidocs.oss-cn-zhangjiakou.aliyuncs.com/res/WgZOZwWGYrMLlLX8/img/0f8ce587-929d-4dc8-a77c-3630361f69e6.png)

$$b\_{1}$$ ，$$b\_{2}$$ 这组正交基可以通过球半径和向量$$\overline{P}_{l}^{cj}$$点乘（内积投影），叉乘（外积垂直）得到：

![image.png](https://alidocs.oss-cn-zhangjiakou.aliyuncs.com/res/WgZOZwWGYrMLlLX8/img/fa351d85-e490-4eef-95e8-187b91c16385.png)
todo: 修改正交基为重投影三维点（即通过待优化状态量转换得到的单位球面投影点）处的标准正交基

残差定义为：单位球面上的观测点和目标点之间的差值在$$b\_{1}$$ ，$$b\_{2}$$ 这组正交基上的投影。

### 雅可比推导：

![image.png](https://alidocs.oss-cn-zhangjiakou.aliyuncs.com/res/WgZOZwWGYrMLlLX8/img/96985174-9245-40de-a210-70c65bf61101.png)

## KB模型量测更新

这里我们需要优化的状态量主要为当前图像时刻的$$R_{imu2world}$$, $$t_{imu2world}$$, 观测为上一帧中的三维点在当前帧被光流跟踪得到的像素坐标$$p_{uv}$$。

我们单独分析一个点$$p_{w}$$, 它在世界系下的坐标为：

$$p_{w}=(x_w,y_w,z_w)$$

转换到当前图像时刻IMU坐标系下的坐标为：

$$p_{imu}=R_{imu2world}^{-1}(p_{w}-t_{imu2world})$$

利用相机和IMU的外参可得，该点在相机坐标系下的三维坐标为：

$$p_{cam}=R_{imu2cam}p_{imu}+t_{imu2cam} = R_{imu2cam}R_{imu2world}^{-1}(p_{w}-t_{imu2world})+t_{imu2cam}$$

将$$p_{cam}$$投影到像素平面：

$$p_{uv}=project(p_{cam})=project(R_{imu2cam}R_{imu2world}^{-1}(p_{w}-t_{imu2world})+t_{imu2cam})$$

从而残差项为：

$$residual_{uv}=project(R_{imu2cam}R_{imu2world}^{-1}(p_{w}-t_{imu2world})+t_{imu2cam}) - cur_{uv}$$

雅可比求解：

对旋转求导：

$$\tfrac{\partial (residual_{uv})}{\partial R_{imu2world}}
=\tfrac{\partial (project(R_{imu2cam}R_{imu2world}^{-1}(p_{w}-t_{imu2world})+t_{imu2cam}) - cur_{uv})}{\partial R_{imu2world}}
\\ = \tfrac{\partial (project(p_{cam}))}{\partial R_{imu2world}} = \tfrac{\partial (project(p_{cam}))}{\partial p_{cam}} * \tfrac{\partial (p_{cam})}{\partial R_{imu2world}}
\\= \tfrac{\partial (project(p_{cam}))}{\partial p_{cam}} *R_{imu2cam}* \tfrac{\partial (R_{imu2world}^{-1}(p_{w}-t_{imu2world})}{\partial R_{imu2world}}$$

已知：$$\tfrac{\partial R^T*P}{\partial \delta\theta} =\lfloor R^T*P\rfloor$$，所以：

$$\tfrac{\partial (residual_{uv})}{\partial R_{imu2world}} 
= \tfrac{\partial (project(p_{cam}))}{\partial p_{cam}} *R_{imu2cam}* (  (R_{imu2world}^{-1}(p_{w}-t_{imu2world}))^T
\\ = \tfrac{\partial (project(p_{cam}))}{\partial p_{cam}} *R_{imu2cam}* \lfloor  p_{imu}\rfloor$$

其中， $$\tfrac{\partial (project(p_{cam}))}{\partial p_{cam}}$$（对应公式不想推了，直接看代码吧），对应代码如下：

对平移求导：

$$\tfrac{\partial (residual_{uv})}{\partial t_{imu2world}} 
=\tfrac{\partial (project(R_{imu2cam}R_{imu2world}^{-1}(p_{w}-t_{imu2world})+t_{imu2cam}) - cur_{uv})}{\partial t_{imu2world}}
\\ = \tfrac{\partial (project(p_{cam}))}{\partial p_{cam}} * \tfrac{\partial (p_{cam})}{\partial t_{imu2world}}
\\= -\tfrac{\partial (project(p_{cam}))}{\partial p_{cam}} *R_{imu2cam}* R_{imu2world}^{-1}$$
