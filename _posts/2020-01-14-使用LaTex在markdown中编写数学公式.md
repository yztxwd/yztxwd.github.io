---
layout:         post
title:          使用LaTex在markdown中编写数学公式
date:           2020-01-14
author_id:  jianyu
toc:        true
math:       true
categories: [Tutorial]
tags: [latex, markdown]
---

## 前言

latex是一种便捷的编写数学公式的语言。之前我在word中编写公式使用的是Mathtype，虽然好用，但是要收费。最近我在学习使用markdown来编写文档，而markdown对latex有着非常棒的支持，这里记录一下使用的方法。

## 基础语法

latex数学公式的引用有两种形式，分别是行内公式和独立公式

- 行内公式：用`$formula$`表示，例如`$\sum_{i=0}^{n}(i^3)$`代表$$ \sum_{i=0}^{n}(i^3) $$
- 独立公式：用`$$formula$$`表示，例如`$$\sum_{i=0}^{n}(i^3)$$`表示$$\sum_{i=0}^{n}(i^3)$$

> 特殊符号的使用需要在前面加上\来转义，包括`#$%&_^~\{}`,比如`$\$$`表示$\$$，对于`\`，可使用`$\backslash$`来表示

## 常用数学表达

### 上下标

- 上标：用`^`后的内容表示上标，例如`$a^b$`表示$a^b$
- 下标：用`_`后的内容表示下标，例如`$a_b$`表示$a_b$
- 混用：例如`$a_{b}^{c}$`表示$a_{b}^{c}$

> 可以使用命令改变上下标的大小（如`\tiny`,`\small`,`\normalsize`,`\large`等），例如`$a_b$`,`$a_{_b}$`,`$a_{\tiny{b}}$`表示$a_{\normalsize{b}}$,$a_{\small{b}}$,$a_{\large{b}}$

### 分数

- `$\frac{}{}$`表示根据环境设置样式的分数，例如`$\frac{y}{x}$`表示$\frac{y}{x}$
- `${}\over{}$`另一种表达方式，例如`${y}\over{x}$`表示${y}\over{x}$

### 根式

- 二次根式：`$\sqrt{expression}$`，例如`$\sqrt{a_b}$`表示$\sqrt{a_b}$
- n次根式：`$\sqrt[n]{expression}$`，例如`$\sqrt[3]{a_b}$`表示$\sqrt[3]{a_b}$

### 省略号

- 用`\dots \cdots \vdot \ddots`表示，`\dots`和`\cdots`的纵向位置不同，例如`$$\dots,\cdots,\vdots,\ddots$$`表示$$\dots,\cdots,\vdots,\ddots$$

### 长公式

`aligned`可以对齐，使用`\\`和`&`来分行和设置对齐位置，例如
```   
$$\begin{aligned}
x=a+b+c \\
d+e+f+g
\end{aligned}$$
```
表示
$$\begin{aligned}
x=&a+b+c \\
& d+e+f+g
\end{aligned}$$

### 分支公式

分支公式通常使用`cases`，例如：
```
$$y=\begin{cases}
    -x,\quad &x \leq 0 \\
    x, &x>0
\end{cases}$$
```
表示
$$y=\begin{cases}
    -x,\quad &x \leq 0 \\
    x, &x>0
\end{cases}$$

### 公式编号

通过命令`\tag{n}`手动为公式编号，例如
```
$$ S(r_k) = \sum_{i=0}^{r_k}\dfrac{\sqrt{a}}{b}\tag{1.1}$$
```

$$ S(r_k) = \sum_{i=0}^{r_k}\dfrac{\sqrt{a}}{b}\tag{1.1}$$

### 上下水平线

- `\overline{expression}`在上方画水平线，例如`$\overline{x+y}$`表示$\overline{x+y}$
- `\underline{expression}`在上方画水平线，例如`$\underline{x+y}$`表示$\underline{x+y}$

### 矩阵

矩阵的每一行以`\\`结束，各个元素之间用`&`分隔，例如：
```
$$\begin{matrix}
x_{_{11} } & x_{_{12} } & \dots & x_{_{1n} } \\
x_{_{21} } & x_{_{22} } & \dots & x_{_{2n} } \\
\vdots & \vdots & \ddots  & \vdots  \\
x_{_{m1} } & x_{_{m2} } & \dots & x_{_{mn} } \\
\end{matrix}$$
```
表示
$$\begin{matrix}
x_{_{11} } & x_{_{12} } & \dots & x_{_{1n} } \\
x_{_{21} } & x_{_{22} } & \dots & x_{_{2n} } \\
\vdots & \vdots & \ddots  & \vdots  \\
x_{_{m1} } & x_{_{m2} } & \dots & x_{_{mn} } \\
\end{matrix}$$

## 常用符号

### 希腊字母

|符号|命令|
|--|--|
|$\alpha$|`$\alpha$`|
|$\beta$|`$\beta$`|
|$\gamma$|`$\gamma$`|
|$\delta$|`$delta$`|
|$\epsilon$|`$\epsilon$`|
|$\zeta$|`$\zeta$`|
|$\eta$|`$\eta$`|
|$\alpha$|`$\alpha$`|
|$\theta$|`$\theta$`|
|$\mu$|`$\mu$`|
|$\lambda$|`$\lambda$`|

## 参考

- [Markdown 中 LaTex 数学公式命令](https://juejin.im/post/5c0a27ee6fb9a049d05d8b70#heading-45)
