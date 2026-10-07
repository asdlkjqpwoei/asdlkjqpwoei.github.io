---
title: Computer Graphics Mathematics
categories: Computer Graphics
date: 2024-05-09
updated: 2026-04-02
---
# Cartesian Coordinate System(笛卡儿坐标系)

使用n个互相垂直的轴描述n维空间，其中的点代表着空间的位置。在高中数学中常用的皆为二维Cartesian坐标系，在游戏行业中，常用的为三维的Cartesian坐标系。

手掌心朝上，无名指和小指蜷缩，食指和拇指伸直并且食指朝向自己身体的前方，中指垂直于手掌，将拇指指向设$x$轴，食指指向设为$y$轴，中指指向设为$z$轴，就是高中数学常用的二维Cartesian坐标系平面加上一条垂直该平面的轴直线。

Game Engine Architecture(游戏引擎架构)书中的右手坐标系图是将食指朝向自己身体上方的图示。

# 矢量运算

## Magnitude(模)

标量，代表矢量在空间中的长度。一般使用勾股定理计算矢量的模。

{% raw %}
$$|a| = \sqrt[]{{a_x}^2 + {{a_y}^2} + {{a_z}^2}}$$
{% endraw %}

### Note

计算平方根通常会损耗性能，尽量使用模的平方。

## Dot Product(点积)

别名：Scalar Product(标量积)或内积(Inner Product)

$$\vec{a} \cdot \vec{b} = {a_x} {b_x} + {a_y} {b_y} + {a_z} {b_z}$$

点积支持数学基本运算中的交换律和分配律。

通常用来对比两个游戏对象的位置和朝向：

* $$\vec{a} \cdot \vec{b} = 1$$

两个对象共线，并且朝向完全相同

* $$\vec{a} \cdot \vec{b} = -1$$

两个对象共线，并且朝向完全相反。

* $$\vec{a} \cdot \vec{b} = 0$$

两个对象的朝向相互垂直。

* $$\vec{a} \cdot \vec{b} > 0$$

两个对象的朝向大致相同。

* $$\vec{a} \cdot \vec{b} < 0$$

两个对象的朝向大致相反。

## Cross Product(叉积)

别名：Vector Product(矢量积)或Outer Product(外积)

只定义于三维空间。结果是一个新的矢量，垂直于原来的两个参与运算的矢量。

$$\vec{a} \times \vec{b} = ({a_y} {b_z} - {a_z} {b_y})\vec{i} + ({a_z} {b_x} - {a_x} {b_z})\vec{j} + ({a_x} {b_y} - {a_y} {b_x})\vec{k} $$

### 叉积的模

$$|\vec{a} \times \vec{b}| = |\vec{a}| |\vec{b}| sin\theta$$

# Linear Interpolation, LERP(线性插值)

用来计算两个已知点的中间点。

$$LERP(\vec{A}, \vec{B}, \beta) = (1 - \beta)\vec{A} + \beta\vec{B} = [(1 - \beta)A_x + \beta B_x, (1 - \beta)A_y + \beta B_y, (1 - \beta)A_z + \beta B_z], \beta \in [0,1]$$

# Matrix(矩阵)

由$m /times n$个标量组成的长方形数组，矩阵可方便的表示线性变化，例如平移、旋转、缩放。

行(Row)与横，列(Column)与竖、纵。

## Special Orthogonal Matrix(特殊正交矩阵), Isotropic Matrix(各向同性矩阵), Orthonormal Matrix(标准正交矩阵)

一种所有行矢量及列矢量均为单位矢量的$3 \times 3$矩阵。

## 变换矩阵

一种可表示三维变换的矩阵，包括平移、旋转、缩放。

## Affine Matrix(仿射矩阵)

能维持直线在变换前后的平行性以及相对的距离比，不一定维持直线在变换前后的绝对长度及角度的$4 \times 4$变换矩阵。

由平移、旋转、缩放、切变所组合而成的变换都是仿射矩阵。

# Euler Angle(欧拉角)

* Pitch(俯仰角)
* Yaw(偏航角)
* Roll(滚动角)

## Gimbal Lock(万向节死锁)

当旋转90度时，三个主轴中的其中一个主轴就会与另一个主轴完全对其，此时三个主轴的状态就叫做万向节死锁，其中两个主轴已经完全对其，无法在单独的围绕其中一个主轴旋转，因为二者已经等效。

# Quaternion(四元数)



# Reference

1. Introduction to Linear Algebra [Fifth Edition] - Gilbert Strang
2. Game Engine Architecture [Third Edition] – Jason Gergory, 叶劲峰译