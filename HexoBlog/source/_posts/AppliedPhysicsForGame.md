---
title: Applied Physics For Game
categories: Applied Physics
date: 2024-05-09
updated: 2026-04-02
---
# Projectile Motion(斜抛运动)

$v$ - 初始速度大小, $t$ - 运动时间, $x$ - 水平位移距离, $y$ - 垂直位移距离, $g$ - 重力加速度, $\theta$ - 物体初始速度方向

Horizontal Displacement(物体水平位移)：

$$ x = v t \cos(\theta) $$

Vertical Displacement(物体垂直位移)：

$$ y = v t \sin(\theta) - \frac{g t ^ 2}{2} $$ 

已知水平位移、垂直位移、初始速度方向，求初始速度大小：

$$ v = \sqrt{\frac{g x^2}{2 x \sin(\theta) \cos(\theta) - 2 y \cos^2(\theta)}} $$

已知水平位移、垂直位移、初始速度大小，求初始速度方向：

$$ \theta = \arctan(\frac{-v^2 \pm \sqrt{v^4 - g (g x^2 + 2 y v^2)}}{-g x}) $$

需用上$\cos^2(theta) = \frac{1}{\tan^2(\theta) + 1}$进行数学推导

水平位移：

$$ d = \frac{v \cos(\theta)(v \sin(\theta) + \sqrt{(v \sin(\theta))^2 + 2 g y})}{\|g}  $$

# Reference

## Wiki

1. [Projectile Motion - Wikipedia](https://en.wikipedia.org/wiki/Projectile_motion)

## Community

1. [抛射体运动在游戏开发中的时间 - 掘金](https://juejin.cn/post/7084128151076339749#heading-27)