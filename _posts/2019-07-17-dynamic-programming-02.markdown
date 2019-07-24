---
layout:     post
title:      "Introduction to Dynamic Programming 02"
subtitle:   "Knapsack Problem"
date:       2019-07-17 00:00:00
author:     "NJL"
header-img: "img/post-bg-2015.jpg"
tags:
    - Dynamic Programming
    - Python
---

### 什么是背包问题 

给定一组物品，每种物品都有自己的重量和价格，在限定的总重量内，我们如何选择，才能使得物品的总价格最高。  

![wiki knapsack](https://upload.wikimedia.org/wikipedia/commons/thumb/f/fd/Knapsack.svg/250px-Knapsack.svg.png)


根据物品能够被选择的次数不同，分为了三类不同问题，有界背包，0-1 背包和无界背包问题。   

我们首先定义：  
$$W$$ : 背包所能承受的总重量     
$$v_i$$ : 第 $$i$$ 个物品的价值  
$$w_i$$ : 第 $$i$$ 个物品的重量  
$$x_i$$ : 选取物品 $$i$$ 多少次  


### 0-1 背包问题 (0-1 knapsack problem)  
每个物品只有两种选择，放入或不放。那么一共有 $$2^n$$ 种选择方法。 将选择的方法画成树的形式，如下图   


$$
\begin{aligned}
&maximize \quad \sum_{i=1}^n v_ix_i \\
&subject\enspace to \quad \sum_{i=1}^nw_ix_i \quad and \quad x_i \in \{0,1\}
\end{aligned}
$$


![dp-01]('/img/in-post/dynamic-problems/dp-01-tree.png')

bounded knapsack problem(BKP)  

$$
\begin{aligned}
&maximize \quad \sum_{i=1}^n v_ix_i \\
&subject\enspace to \quad \sum_{i=1}^nw_ix_i \quad and \quad 0 \le x_i \le c
\end{aligned}
$$

unbounded knapsack problem (UKP)  

$$
\begin{aligned}
&maximize \quad \sum_{i=1}^n v_ix_i \\
&subject\enspace to \quad \sum_{i=1}^nw_ix_i \quad and \quad x_i \ge 0
\end{aligned}
$$