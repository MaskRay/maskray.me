---
layout: post
title: 並行N-body模擬
author: MaskRay
tags: [parallel, algorithm, n-body]
---

## 緣由

和專業有關係的第一門有趣的課結束了，覺得有必要記錄些什麼。這篇文章就用來紀念我覺得最有意義的一次作業。

## 要求

需要分別編寫MPI、Pthreads、OpenMP的並行實作。

## 層次

我不願意把代碼重複三份，分別爲三種並行庫編寫程式，所以就用 autotools 管理項目，用不同的選項來啓用不同的功能。這樣做的另一個好處是很容易支持MPI+Pthreads或者MPI+OpenMP甚至是MPI+Pthreads+OpenMP，儘管這樣混用不會帶來性能上的提升。

<!-- more -->

## 算法

### 加速度計算

根據牛頓萬有引力公式和第二運動定律很容易導出直觀的 $O(n^2)$ 算法。使用 Barnes-Hut 近似模擬算法可以優化至 $O(n \log n)$。代碼重用做得不是很好，爲2d和3d情形分別實現了quad-tree和oct-tree。

According to Newton's law of universal gravitation, every body attracts every other body
by a force pointing along the line intersecting both centers.
The force is proportional to the product of the two masses and inversely proportional to
the square of the distance between them:

\begin{eqnarray}
F = G \frac{M m}{r^2}
\end{eqnarray}

By Newton's Second Law, the acceleration is expressed by $F = m a$.

Combing the two laws, we can easily deduce an direct-sum algorithm to calculate the accelerations
of these bodies, which would be $O(n^2)$. However, the brute-force simulation
has low data dependency and is easy to be reformed to a parallelized algorithm.

Barnes-Hut simulation decreases the time complexity to $O(n \log n)$
which exploting a quad-tree (in the 2-dimensional case) and
approximate a distant cluster of bodies to a point.  Each node in the
quad-tree represents a subspace. The root node represents the whole
space and its four children represent the four quadrants of the space.
The resultant force attracted by a distant cluster of bodies is treated as
being attracted by the center of mass.

To determine if the cluster is sufficiently far away, I compute the
quotient $s/d$, where $s$ is the size of the subspace and $d$ the
distance between the body and the center of mass. If $s/d < \theta$,
the cluster is considered distant.


### 碰撞檢測

這裏是我實現得最複雜的部分。如果要計算出每個碰撞是非常耗時的，實踐中不得不妥協使用“每個球只允許主動碰撞一個球，不處理其他的”
之類的辦法。

單單考慮碰撞檢測，這類問題通常會把球用 axis-aligned minimum bound-ing rectangle 包裹起來。
相當一部分算法能處理這個問題，比如 R-tree、BSP tree、2-dimensional segment tree、spatial hashing、掃描線+interval tree、
掃描線+segment tree等。我在項目中實現了後兩種以及 BSP tree。

Each body along with the imaginary position when it reaches the
destination after the time slice, is enclosed in a *axis-aligned
minimum bounding rectangle*. It is easy to decuce that if two body
collides in the time slice, then their bounding rectangles
intersect. The property can be used to accelerate the performance of
collision detecting.

Several existing approaches deal with the intersection detection problem straightforwardly.

- R-tree and its variants. R-trees are tree data structures used for
  spatial access methods and the most immediate solution to the problem.
- Binary space partitioning tree. They are developed in the context of computer graphics
  are are also suitable for the problem.
- 2-dimensional segment tree. First notice that two rectangles overlaps iff their projections
  on coordinate axes overlap. We need to construct a 2-dimensional segment tree with $O(n \log^2 n)$
  space which supports $O(n \log^2 n)$ query.
- Spatial hashing. Subdivide the plane into a set of voxels (a term used in image processing,
  the smallest distinguishable box-shaped part) and thus forms a mesh. Each voxel contains a
  bucket of bodies.  Put each body into the buckets corresponsing its
  occupied voxels. Before insertng into a voxel, each existing body in the
  cell is a candidate for potential collision.

Other approaches including using a sweep line algorithm to reduce the
problem to the 1-dimensional case (overlapping interval search
problem).

- Interval tree. It is a tree data structure holding intervals. Space
  complexity $\Theta(n)$, construction time complexity $O(n \log n)$,
  query time complexity $O(\log n + k)$.
- Segment tree. Space/construction time complexity $O(n \log n)$,
  query time complexity $O(\log n + k)$.

Approaches exploiting the property of temporal coherence:

- TPR-tree and a few ramification. A variant of R-tree taking
  advantages of spatio-temporal updates.

I'll expatiate three data structures implemented in my project.

#### Binary space partitioning tree

The implementation is in the file =src/BSPTree.cc=.

The construction process is rather simple and straightforward.
I start with the whole scene and a list of bounding boxes, recursively
divide the scene into two until the number of bodies in the subspace
is less than a threshold (=NODE_SIZE_THRESHOLD=) or there is no
balanced partition scheme. Each separating line is parallel to X, Y or Z axis.

For a range query, we examine whether the cube in question intersects
with the left-subspace.  If so, recursively do a query in the
left-subspace. The same process also applies to the right-subspace.

#### Sweep line algorithm + Interval tree

The implementation is in the file =src/IntervalTree.cc=.

Well, in this project I also use a much easier but also efficient
approach combining a sweep line algorithm and an interval tree besides
a BSP tree implementation.

- Each bounding rectangle has an upper and a lower edges. They mark the timestamp between which the
  rectangle is valid.
- Sort these edges by their y coordinates and process these edges in
  ascending order.  For an upper edge, search all the valid rectangles
  and find those x-axis spans (intervals) which overlaps the span of the
  current rectangle, and then insert the x-axis span into a data
  structure. For these overlapping intervals, check whether the
  corresponding rectangle is marked valid and if so, do a precise
  detection.  For a lower edge, delete the x-axis span from the data
  structure.

The unamed data structure should support following operations
efficiently: insert an interval, query all intervals overlapping given
interval (overlapping interval search problem).

The query can be fulfilled by an interval tree. Construct an
interval tree with all intervals but do not really store these
intervals. This makes the constructed tree balanced.

Suppose $n$ is the number of bodies. The interval tree can be
constructed in $\Theta(n \log n)$ time while each insertion runs in
$O(\log n)$ time and each query runs in $O(\log n + k)$ time where $k$
is the number of overlapping intervals.

#### Sweep line algorithm + Segment tree

Ditto on the reduction part. Create a sorted list of unique X coordinates
extracted from those bounding boxes: $p_0, p_1, p_2, \ldots, p_{m}$.
These coordinates form $m$ segments: $[p_0,p_1), [p_1,p_2), \ldots, [p_{m-1},p_m)$.
Each leaf node represents a segment while each internal node represents several segments.

### 碰撞模擬

如果當前幀兩球會碰撞，那麼把它們瞬移至時間片中碰撞的位置再改變它們的速度。爲了防止兩球粘在一起，
我會把它們拉開到之前的某個時刻剛剛接觸時處理碰撞。

The most accurate approach is to deduce the nearest collision event,
simulate the event, and then find another. However, this is incapable
of handling hundreds of bodies, especially when there are some
high-speed and light bodies. They will frequently change directions
and affect other bodies.

A comparatively coarse one is to calculate $n(n-1)/2$ pair-wise collision events
and simulate them one after another. It is coarse in the sense that it may not
handle multiple collisions taking place on one single body correctly.

The two mentioned ways are neither efficient nor parallelizable. One simple mend
making the latter parallelizable is to update velocities after all collision events are
processed, but incurs the inaccuracy that the updated velocity of the other body in a collision won't
be reflected immediately.

I used a non-parallelizable, asymptotically optimal sequential approach that is a bit accurate then the above one.

By the way, to make the calculation of the collision point more accurate, I'll use a techinique stated below:

Suppose $\vec{p_0}, \vec{p_0}$ are positions of the two bodies and $\vec{\Delta p}, \vec{\Delta q}$
their translational vectors. Then we can parametrize their positions:

\begin{eqnarray}
\vec{p}(t) = \vec{p_0} + t \vec{\Delta p} \\
\vec{q}(t) = \vec{q_0} + t \vec{\Delta q}
\end{eqnarray}

We will try to solve the equation $|\vec{p}(t)-\vec{q}(t)|=R+r$ where $R,r$ are radii of the two bodies.
This is a quadratic equation and we should care whether the smaller root $t_0$ satisfies $0 \le t_0 \le 1$ or the two bodies overlap.
If so, teleports the two bodies to their collision moment which is $now + t_0 \Delta t$ and carry out an elastic collision.

Since collision only imparts force along the line of collision, only the
velocities that are along the line of collision should be changed, thus degrades to
1-dimensional case.

## 截圖

原始碼包中 scripts/a.py 是用 pyopengl 編寫的簡單播放器。
可以用

    src/n-body -f text -d 3 -n 20 --fps 20 -t 2 | python scripts/a.py

命令播放。

![截圖](/static/n-body.jpg)

## 原始碼

[在github上](https://github.com/MaskRay/n-body)
