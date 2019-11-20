---
layout: single
toc: true
title:  "Golomb Coding"
date:   2019-11-20 00:00:00 +0800
categories: tutorial
---

I was investigating several data compression techniques and came across Golomb coding. Golomb coding is very simple to understand, but is only suitable for data that matches a certain distribution.

## Simplified Overview

Here, we assume that we are encoding a sequence of unsigned integers. The technique to extend the coding to signed integers will be explained later.

For each integer (expressed in binary) to be encoded, we split it into two parts: $$w$$ bits to the right, and the rests to the left. For example to encode $$23 (10111_2)$$ with $$w=3$$,

$$10111_2\rightarrow 10_2 \mid 111_2$$

The left part of value $$p$$ is encoded using $$p$$ number of $$0$$s and followed by a $$1$$ (unary code). From the example, we get $$(10_2=2)\rightarrow 001_2$$. The right part is simply appended to it and get $$001111_2$$. That's all! 

## Parameter

Noticed that splitting a number $$N$$ with $$w$$ can be expressed as dividing the number by $$M = 2^w$$:

$$N = qM+r$$

where $$0\leq r < M$$. Then the quotient $$q$$ is the left part and the remainder $$r$$ is the right part. Using this equation, we realise that the parameter can be of any numbers, and not just a power of two. In fact, a parameter with a power of two is a special case of Golomb coding called Rice coding.

## Truncated Binary Encoding

Let's say we pick $$M=10$$. To encode 32, we should have

$$33 = 3\times 10 + 2\rightarrow 0001_2 \mid 0010_2$$

Why do we write $$0010_2$$ for the right part, instead of $$10_2$$, you may ask? The reason is for the unambiguity when interpreting. If they have variable length, you can interpret $$001101000010010$$ as either $$[001\vert 101][00001\vert 0010]$$ or $$[001\vert 1010][0001\vert 0010]$$ and you won't be able to differentiate. Hence, it seems to be the case where the right part must have the same length, which equals to $$\lceil \log_2(M)\rceil$$ bits, but there is a smart way to save some bits!

Let's look at the possible values for the right part.

| $$r$$   | binary    |
|-------  |---------- |
| $$0$$   | $$0000$$  |
| $$1$$   | $$0001$$  |
| $$2$$   | $$0010$$  |
| $$3$$   | $$0011$$  |
| $$4$$   | $$0100$$  |
| $$5$$   | $$0101$$  |
| $$6$$   | $$0110$$  |
| $$7$$   | $$0111$$  |
| $$8$$   | $$1000$$  |
| $$9$$   | $$1001$$  |

Observe that most of the first bit is $$0$$, so we really want to omit that. However, we still need to find a way to identify $$8$$ and $$9$$. We can use this strategy - set some numbers to signalise that we need to read the next (4-th) bit, and if the next bit is a 1, it refers to one of $$8$$ or $$9$$.

For example, we can let $$110$$ and $$111$$ to be the special number. When we read $$110$$ or $$111$$, we read the 4-th bit and make $$1101$$ refers to $$8$$ and $$1111$$ refers to $$9$$. We denote this as

$$\begin{align}\color{red}{110}&\rightarrow 6(\color{red}{110}0),8(\color{red}{110}1) \\ \color{blue}{111}&\rightarrow 7(\color{blue}{111}0),9(\color{blue}{111}1)\end{align}$$

| $$r$$   | binary    | truncated binary  |
|-------  |---------- |------------------ |
| $$0$$   | $$0000$$  | $$000$$           |
| $$1$$   | $$0001$$  | $$001$$           |
| $$2$$   | $$0010$$  | $$010$$           |
| $$3$$   | $$0011$$  | $$011$$           |
| $$4$$   | $$0100$$  | $$100$$           |
| $$5$$   | $$0101$$  | $$101$$           |
| $$6$$   | $$0\color{red}{110}$$   | $$\color{red}{1100}$$           |
| $$7$$   | $$0\color{blue}{111}$$  | $$\color{blue}{1110}$$          |
| $$8$$   | $$1000$$   | $$\color{red}{1101}$$           |
| $$9$$   | $$1001$$   | $$\color{blue}{1111}$$          |

Using this mapping, we have saved 6 unnecessary $$0$$ bits! Turns out that we can also order it in a different way such that it is easier to compute

$$\begin{align}\color{red}{110}&\rightarrow 6(\color{red}{110}0),7(\color{red}{110}1) \\ \color{blue}{111}&\rightarrow 8(\color{blue}{111}0),9(\color{blue}{111}1)\end{align}$$

| $$r$$   | binary                  | truncated binary        | decimal   |
|-------  |------------------------ |------------------------ |---------  |
| $$0$$   | $$0000$$                | $$000$$                 | $$0$$     |
| $$1$$   | $$0001$$                | $$001$$                 | $$1$$     |
| $$2$$   | $$0010$$                | $$010$$                 | $$2$$     |
| $$3$$   | $$0011$$                | $$011$$                 | $$3$$     |
| $$4$$   | $$0100$$                | $$100$$                 | $$4$$     |
| $$5$$   | $$0101$$                | $$101$$                 | $$5$$     |
| $$6$$   | $$0\color{red}{110}$$   | $$\color{red}{1100}$$   | $$12$$    |
| $$7$$   | $$0\color{blue}{111}$$  | $$\color{red}{1101}$$   | $$13$$    |
| $$8$$   | $$1000$$                | $$\color{blue}{1110}$$  | $$14$$    |
| $$9$$   | $$1001$$                | $$\color{blue}{1111}$$  | $$15$$    |

Now, we are ready to create a generalised formula for the truncated binary of the right part. Let $$b = \lfloor \log_2(M)\rfloor$$. The number of $$r$$ that starts with a $$0$$ bit is $$2^b$$ while the number of $$r$$ that starts with a $$1$$ bit is $$M - 2^b$$. Among the $$2^b$$ numbers that starts with $$0$$, we need $$M-2^b$$ of them to be the special numbers that signal us to read an additional bit. Therefore, the number of $$r$$ that can be truncated directly to $$b$$ bits is $$2^b - (M-2^b) = 2^{b+1}-M$$. We call this the _threshold_.

From the example above, $$b = \lfloor \log_2(10)\rfloor = 3$$ and the threshold is $$2^{b+1}-M=2^4-10=6$$. This means that the numbers from $$0$$ to $$5$$ can be truncated to using only $$b=3$$ bits.

For the $$r$$ values that are outside of the threshold (the first one being $$2^{b+1}-M$$), we noticed that it has an offset of $$2^{b+1}-M$$. Hence, these values are stored as $$r+2^{b+1}-M$$ using $$b+1$$ bits.

Again using the example above, the offset is $$2^{b+1}-M=6$$, hence $$6$$ is truncated to $$6+6=12$$, $$7$$ to $$7+6=13$$ etc.

As an exercise, verify that with $$M=10$$, the compressed code

$$000101011110001101011111$$

refers to the array $$[32,8,25,19]$$.
