---
title: Game Engine Architecture
categories: Game Engine
---
# Runtime Engine Architecture

## Target Hardware(硬件层)

代表执行游戏的计算机系统或者主机，例如PC、XBox、PlayStation。

## Device Drivers(驱动程序)

负责管理硬件资源，一般由操作系统或者硬件开发商提供。

## Operating System(操作系统)

## Third-Party SDKs and Middleware(第三方软件开发包和中间件)

引擎一般都会借用一部分第三方组件，例如DirectX、OpenGL等图形库，又或者Boost等数据结构及算法库，还有Havok、PhysX等负责物理的开发库。

## Platform Independence Layer(平台独立层)

将平台有关的部分与硬件底层分离，通常会有平台检测，包装平台相关的开发库和基础API，保证接口在不同平台的行为一致。

## Core System(核心系统)

主要包含开发时需要用到的使用软件和功能，例如断言、内存分配、数学库和自定义的数据结构及算法等。

## Resource Manager(资源管理器)

负责访问和管理任何类型的游戏资产和其他输入数据。

## Redering Engine(渲染引擎)

渲染引擎没有特别的架构，但是一般通用来说都采用分层架构

### Low-Level Renderer(低阶渲染器)

包含全部原始的渲染功能，着重渲染丰富的几何图元。

### Scene Graph/Culling Optimization(剔除优化)

在低阶渲染器提交的几何图形不考虑其是否可见，在该层将不可见的图形剔除。

### Visual Effect, VFX(视觉效果)

一般有粒子系统、贴花系统、光照贴图及环境贴图、动态阴影、全屏后期处理效果等。

### Front End(前端)

一般就是指二维的界面了，例如抬头显示器、菜单等等。

## Profiling and Debugging Tools(性能剖析及调试工具)

进行性能剖析以进行性能优化，否则市面上大多数硬件都无法负担游戏带来的负载从而极大地影响游戏体验。

## Collision and Physics(物理，刚体动力学模拟)

提供拟真的物理模拟

## Animation(动画)

负责提供动画相关的功能。

## Human Interface Devices(人体学接口设备)

负责处理玩家输入，输入可以有多种设备提供，例如：键盘和鼠标、游戏手柄、方向盘、遥控器等。

## Audio(音频)

提供处理及优化音频的功能，以实现更好的声乐效果。

## Only Multiplayer/Networking(在线多人/网络)

提供多人游戏的功能

## Gameplay Foundation Systems(游戏性基础系统)

游戏性(Gameplay)：游戏内进行的活动、支配游戏虚拟世界的规则、玩家角色的能力、其他角色和对象的能力、玩家的长短目标。

## Game-Specific Subsystems(专用子系统)

针对游戏性进行特制开发的各类子系统，如专用渲染和AI。

# Reference

* Game Engine Architecture [Third Edition] – Jason Gergory, 叶劲峰译