---
layout:     post
title:      "Introduction to Dynamic Programming 01"
subtitle:   ""
date:       2019-07-14 00:00:00
author:     "NJL"
header-img: "img/post-bg-2015.jpg"
tags:
    - Dynamic Programming
    - Python
---


### What is Dynamic Programming

动态规划其实本质上和分治法相类似，通过组合子问题的解来求解原问题。分治法将问题划分为***互不相交***的子问题，递归求解并将它们组合起来。动态规划用于***子问题重叠***的情况，对每个子问题只求解一次并将结果保存在表格中(dp数组)，避免不必要的重复计算。  

用动态规划求解问题，一般分为以下几个步骤：

1. 刻画最优解的结构特征
2. 递归定义最优解的值
3. 计算最优解的值，通常采用自底向上方法
4. 利用计算出的信息构造最优解


下面通过 leetcode 的一个题目来实践一下

####  [leetcode coin-change](https://leetcode.com/problems/coin-change/)
给定一把硬币，面值分别为1元，3元和5元。要购买一个价值为N的物品，怎样选取硬币才能使用最少的硬币刚好能购买到物品（硬币和=物品价值）。每个硬币可以使用多次。  

首先定义 $$dp[n]$$ 为物品价值为$$n$$时需要的最少硬币数。如果存在选取方法，那么最后一个的硬币有以下三种选取方式, 选择 1，3 或者 5


最后一个硬币的面值 | 剩余价值 | 构造剩余价值所需最少硬币数
---|---|---
1 | n-1| dp[n-1]
3| n-3 | dp[n-3]
5 | n-5 | dp[n-5]

根据上表可以轻松的找出该问题的最优解的结构特征，$$dp[n]$$ 必定是三种方法中最小的那个

$$dp[n] = min\{dp[n-i], i\in\{1,3,5\}\} + 1$$

需要注意的是 $$dp[n]$$ 的求解依赖于前几次的解。  

到这里我们完成了前两步，接下来就是计算过程

##### bottom-up and top-down

在推导出最优子结构的公式之后，一般有两种方式来求解。自顶向下和自底向上。

自顶向下，通过递归和备忘机制。    
自底向上，从最小（初）的子问题开始求解。    


> 自顶向下和自底向上的优劣可参考 https://stackoverflow.com/questions/6164629/what-is-the-difference-between-bottom-up-and-top-down  

当价值为0时，硬币数肯定为0，而且选取时硬币价格不能大于物品价格。 例如物品价格为3时，不能选取硬币5，  

 $$dp[3]=min(dp[3-3], dp[3-1]) + 1$$
 
 最终变成比较 $$dp[2]$$ 和 $$dp[0]$$ 哪个更小

 $$min(dp[0], dp[2]) + 1 = dp[0] + 1$$

以下是详细的自底向上的计算过程，

物品价值`n` | 最少硬币数 `dp[n]` | 计算方法
---|--- | --- | ---
0 | dp[0] = 0 |
1 | dp[1] = 1 |  min(dp[1-1]) + 1 == dp[0] + 1
2 | dp[2] = 2 |  min(dp[2-1]) + 1 == dp[1] + 1
3 | dp[3] = 1 |  min(dp[3-3], dp[3-1]) + 1 == dp[0] + 1
4 | dp[4] = 2 |  min(dp[4-3], dp[4-1]) + 1 == dp[1] + 1
5 | dp[5] = 1 |  min(dp[5-5], dp[5-3], dp[5-1]) + 1 = dp[0] + 1
6 | dp[6] = 2 |  min(dp[6-5], dp[6-3], dp[6-1]) + 1 = dp[1] + 1
7 | dp[7] = 3 | min(dp[7-5], dp[7-3], dp[7-1]) + 1 = dp[2] + 1
8 | dp[8] = 2|  min(dp[8-5], dp[8-3], dp[8-1]) + 1 = dp[3] + 1
9 | dp[9] = 3 |  min(dp[9-5], dp[9-3], dp[9-1]) + 1  = dp[4] + 1
10 | dp[10] = 2|  min(dp[10-5], dp[10-3], dp[10-1]) + 1 = dp[5] + 1
11 | dp[11] = 3 | min(dp[11-5], dp[11-3], dp[11-1]) + 1 = dp[6] + 1
    

top_down_solution：自顶向下的纯递归方法，看起来直观符合之前的公式定义，但会导致 TLE 错误，因为有大量的重复计算。

~~~ python
class Solution(object):        
    # coins = [1,3,5] amount=11
    
    MAX_VALUE = pow(2, 32) - 1    
    def coinChange(self, coins, amount):
        top_down_solution
        min_val = self.top_down_solution(coins, amount)
        return min_val if min_val != Solution.MAX_VALUE else -1
        
    # 递归， 会导致 Time Limit Exceed

    def top_down_solution(self, coins, amount):
        if amount < 0:
            return Solution.MAX_VALUE
        if amount == 0:
            return 0
        min_val = min([self.top_down_solution(coins, amount - coin) for coin in coins])
        if min_val != Solution.MAX_VALUE:
            return min_val + 1
        else:
            return min_val
~~~


top_down_optimized_solution：带备忘的递归，将已计算过的结果保存下来，避免重复计算。　　

~~~ python
class Solution(object):
    # coins = [1,3,5] amount=11
    
    MAX_VALUE = pow(2, 32) - 1
    def coinChange(self, coins, amount):
        # top down optimized solution
        dp = [None] * (amount + 1)
        dp[0] = 0
        self.top_down_optimized_solution(coins, amount, dp)
        return dp[amount] if dp[amount] != Solution.MAX_VALUE else -1
        
    #　递归带备忘

    def top_down_optimized_solution(self, coins, amount, dp):
        if amount < 0:
            return Solution.MAX_VALUE

        if dp[amount] is not None:
            return dp[amount]
    
        min_val = min([self.top_down_optimized_solution(coins, amount - coin, dp) for coin in coins])
        if min_val != Solution.MAX_VALUE:
            dp[amount] = min_val + 1
        else:
            dp[amount] = min_val
        return dp[amount]
~~~

bottom_up_solution：自底向上，从最小的子问题开始求解，当计算某个问题时，它所依赖的子问题都已经求解完了。    　　

~~~ python
class Solution(object):
    # coins = [1,3,5] amount=11   

    MAX_VALUE = pow(2, 32) - 1
    def coinChange(self, coins, amount):
        # bottom up solution
        min_val = self.bottom_up_solution(coins, amount)
        return min_val if min_val != Solution.MAX_VALUE else -1

    # 自底向上  

    def bottom_up_solution(self, coins, amount):
        dp = [Solution.MAX_VALUE] * (amount + 1)
        dp[0] = 0
        for value in range(1, amount + 1):
             min_val = Solution.MAX_VALUE
            for coin in coins:
                if value - coin >= 0 and dp[value-coin] < min_val:
                       min_val = dp[value - coin]
            dp[value] = min_val + 1 if min_val != Solution.MAX_VALUE else min_val
        return dp[amount]
~~~
