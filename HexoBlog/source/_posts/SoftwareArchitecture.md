---
title: Software Architecture
categories: Software Architecture
---
当自己尝试做过小工具或者参与过一些大项目，会发现如果代码没有进行合理的组织和分配，会变得极难维护，而且会降低自己对开发工作的动力，最终不了了之。

# Observer Pattern(观察者模式)

一种行为型设计模型，一对多的依赖关系。当一个对象的状态发生改变时，其所有依赖者都会收到通知并自动更新。

# Model View Controller(模型-视图-控制器, MVC)

目的是实现一种动态的程序设计，一般在交互式应用程序中有所体现。

Model(模型): 一般负责程序的逻辑和数据的处理，提供操作。

View(视图): 一般是负责如何显示或表示模型，反馈。

Controller(控制器): 一般是负责处理用户输入事件，对模型进行操作，控制程序工作流程。

# Entity Component System(实体-组件-系统, ECS)

ECS遵循组合优于继承的原则，每一个实体不由类继承所定义，但是会通过组件相互关联，系统在全局范围内对有所需组件的实体进行操作。

Entity(实体): 通常是指一些通用对象，比如游戏开发中，每个游戏对象都是实体，内有一个唯一的标识符。

Component(组件): 通常是指实体所拥有的特定功能或者某一个方面，并且拥有相应的数据。

System(系统): 通常是指一个功能过程，作用于具有所需组件所有实体的过程，比如物理系统查询具有质量组件的实体。

# 除了常规遇到的架构，还有更多小巧思吗？

1. 子类
2. 组合优于继承
3. 将动作定义成一个类
4. 使用实体元类(？)

# Reference

1. [Bob Nystrom - Is There More to Game Architecture than ECS?](https://www.youtube.com/watch?v=JxI3Eu5DPwE)
2. [Entity Component System - wikipedia](https://en.wikipedia.org/wiki/Entity_component_system)
3. [观察者设计模式](https://refactoringguru.cn/design-patterns/observer)