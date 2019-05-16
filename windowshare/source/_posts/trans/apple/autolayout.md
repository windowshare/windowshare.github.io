---
title: 自动布局指南(AutoLayoutGuide)
date: 2019.05.13 20:37:55
categories:
- 翻译
- apple
tags:
- autolayout
---

自动布局指南



# 开始

## 了解自动布局

> 重要
>
> 本文档将不再更新，关于Apple SDKs最新的信息，请访问[文档网站](https://developer.apple.com/documentation).

自动布局通过视图中放置的约束，动态计算视图层级中所有视图的位置和大小。例如，您可以约束一个按钮，使其与图像视图水平居中，上边缘始终保持在图像底部下方8个像素。如果图像的大小或位置改变，按钮的位置将自动调整匹配。

这种基于约束的设计方法允许你构建能够动态响应内部和外部变化的用户界面。

### 外部变化

当父视图的大小或者形状的发生改变时，会发生外部改变。每次更改时，都必须更新视图层次结构的布局，以便最好地利用可用空间。以下是外部变化的一些常见来源：

- 用户调整窗口大小 (OS X)。
- 用户在iPad上进入或离开分屏视图 (iOS)。
- 设备旋转 (iOS)。
- 来电提示条和语音录制条出现或消失 (iOS)。
- 您想支持不同大小类型。
- 您想支持不同屏幕大小。

大多数这些改变都在运行时发生，他们需要您的应用程序动态响应。其他，如支持不同屏幕大小，代表您的应用程序适配不同的环境。即使屏幕大小通常不会再在运行时发生改变，也需要创建自适应界面让您的应用程序在iPhone 4S，iPhone6 Plus，甚至iPad上面都能同样良好地运行。自动布局也是iPad上支持幻灯片和分屏视图的关键组件。



### 内部变化

当用户界面中的视图或控件的大小发生改变时，会发生内部改变。

以下是内部变化的一些常见来源:

- 应用程序显示的内容发生变化。
- 应用程序支持国际化。
- 应用程序支持动态字体 (iOS).

当您的应用程序内容变化时，新的内容可能需要与旧的不同的布局。这种情况常常发生在应用程序的文本和图片的显示上。例如，一个新闻应用需要根据各个新闻文章的大小调整其布局。同样地，照片拼贴必须处理各种图像的尺寸和宽高比。

国际化是使您的应用能够适应不同语言，地区和文化的过程。 国际化应用程序的布局必须考虑到这些差异，并在应用程序支持的所有语言和区域中正确显示。

国际化对布局有三个主要影响。 首先，当您将用户界面翻译成其他语言时，标签需要不同的空间量。 例如，德语通常比英语需要更多的空间。 日语经常要求更少的空间。

其次，用于表示日期和数字的格式可以在不同地区之间变化，即使语言没有变化。 虽然这些更改通常比语言更改更微妙，但用户界面仍需要适应大小的微小变化。

第三，改变语言不仅会影响文本的大小，还会影响布局的组织。 不同语言使用不同的布局方向。 例如，英语使用从左到右的布局方向，而阿拉伯语和希伯来语使用从右到左的布局方向。 通常，用户界面元素的顺序应与布局方向匹配。 如果按钮位于视图的右下角，则阿拉伯语应该是视图的左下角。

最后，如果您的iOS应用支持动态字体，则用户可以更改应用中使用的字体大小。 这可以更改用户界面中任何文本元素的高度和宽度。 如果用户在您的应用程序运行时更改字体大小，则字体和布局都必须适应。



### 自动布局是基于Frame的布局

布局用户界面有三种主要方法。

1.可以编程方式布局用户界面

2.可以使用自动调整大小来自动执行对外部更改的一些响应。

3.可以使用自动布局

传统上，应用程序通过以编程方式为视图层次结构中的每个视图设置Frame来布局其用户界面。 Frame在superview的坐标系中定义了视图的原点，高度和宽度。

![image: ../Art/layout_views.pdf](https://developer.apple.com/library/archive/documentation/UserExperience/Conceptual/AutolayoutPG/Art/layout_views_2x.png)

要布置用户界面，必须计算视图层次结构中每个视图的大小和位置。 然后，如果发生了改变，则必须重新计算所有受影响视图的帧。

在许多方面，以编程方式定义视图的Frame提供了最大的灵活性和功能。 当发生变化时，您可以根据需要进行任何更改。 但是，由于您还必须自己管理所有更改，因此布置简单的用户界面需要花费大量精力进行设计，调试和维护。 创建真正的自适应用户界面会使难度增加一个数量级。

您可以使用自动大小遮罩来帮助减少这些工作。 自动大小遮罩定义了视图的Frame在其父视图的Frame发生更改时的更改方式。 这简化了适应外部变化的布局的创建。

但是，自动大小遮罩支持相对较小的可能布局子集。 对于复杂的用户界面，通常需要使用自己的程序更改来扩充自动大小遮罩。 此外，自动大小遮罩仅适应外部变化。 它们不支持内部更改。

虽然自动大小遮罩只是程序布局的迭代改进，但自动布局代表了一种全新的范例。 您不必考虑视图的Frame，而是考虑它的关系。

自动布局使用一系列约束来定义用户界面。 约束通常表示两个视图之间的关系。 然后，“自动布局”会根据这些约束计算每个视图的大小和位置。 这会生成动态响应内部和外部更改的布局。

![image: ../Art/layout_constraints.pdf](https://developer.apple.com/library/archive/documentation/UserExperience/Conceptual/AutolayoutPG/Art/layout_constraints_2x.png)

用于设计一组约束以创建特定行为的逻辑与用于编写过程代码或面向对象代码的逻辑非常不同。 幸运的是，掌握自动布局与掌握任何其他编程任务没有什么不同。 有两个基本步骤：首先，您需要了解基于约束的布局背后的逻辑，然后您需要学习API。 在学习其他编程任务时，您已成功执行了这些步骤。 自动布局也不例外。

本指南的其余部分旨在帮助您轻松过渡到自动布局。[ “无约束的自动布局”](https://developer.apple.com/library/archive/documentation/UserExperience/Conceptual/AutolayoutPG/AutoLayoutWithoutConstraints.html#//apple_ref/doc/uid/TP40010853-CH8-SW1)一章介绍了一种高级抽象，它简化了自动布局支持的用户界面的创建。 [“约束剖析”](https://developer.apple.com/library/archive/documentation/UserExperience/Conceptual/AutolayoutPG/AnatomyofaConstraint.html#//apple_ref/doc/uid/TP40010853-CH9-SW1)一章提供了您需要了解的背景理论，以便您自己成功地与自动布局进行交互。 在["Interface Builder中使用约束"](https://developer.apple.com/library/archive/documentation/UserExperience/Conceptual/AutolayoutPG/WorkingwithConstraintsinInterfaceBuidler.html#//apple_ref/doc/uid/TP40010853-CH10-SW1)描述了用于设计自动布局的工具，并且[编程创建约束](https://developer.apple.com/library/archive/documentation/UserExperience/Conceptual/AutolayoutPG/ProgrammaticallyCreatingConstraints.html#//apple_ref/doc/uid/TP40010853-CH16-SW1)和[自动布局技巧大全](https://developer.apple.com/library/archive/documentation/UserExperience/Conceptual/AutolayoutPG/AnatomyofaConstraint.html#//apple_ref/doc/uid/TP40010853-CH9-SW1)章节详细描述了API。 最后，["自动布局技巧大全"](https://developer.apple.com/library/archive/documentation/UserExperience/Conceptual/AutolayoutPG/LayoutUsingStackViews.html#//apple_ref/doc/uid/TP40010853-CH3-SW1)提供了各种复杂程度的样本布局，您可以在自己的项目中学习和使用，调试自动布局提供建议和工具，以便在出现任何问题时修复问题。


## 没有约束的自动布局

## 约束剖析

## 在Interface Builder中使用约束



# 自动布局技巧大全

## 堆栈视图

## 简单约束

## 具有固定内容大小的视图



# 调试自动布局

## 错误类型

## 不满意的布局

## 模棱两可的布局

## 逻辑错误

## 调试技巧和提示



# 高级自动布局

## 以编程方式创建约束

## 特定大小类布局

## 使用滚动视图

## 使用自定义表视图单元

## 改变约束



# 附录

## 视觉格式化语言(VFL)



# 修改记录

## 文档修改记录



