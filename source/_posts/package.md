---
title: package
date: 2016-10-09 20:34:29
tags: Knapsack problem
---
&emsp;&emsp;问题：假设银行中有若干的存款，存款额为正整数，例如1，3，5，5。。。1000。。。。现在需要放贷N，N也为正整数。请问怎么样才能让若干的存款额相加尽量接近贷款额。<!--more-->该问题可以抽象为：有若干个正整数，例如1，3，5，5。。。。。每个数都有若干个，给定一个正整数N，怎么样才能用若干个数相加尽量接近正整数N。一个简单的思路是，我们可以先把给定的数字从小到大排序，然后从最大的开始取。如果当前的sum<N,则继续取下一个。该方案的时间复杂度为O（nlgn）虽然简单，但是误差较大。例如有数字60，60，100，给定的N的为120，按照改算法结果为100，误差为20。其实换个思路，该问题其实是一个多重背包问题。有N件物品，每个物品的大小为C,价值为W，背包的大小为V，对每一个物品都有放和不放两个状态。
&emsp;&emsp;对于简单的01背包问题，有N件物品放入一个容量为V的背包中，放入第i件物品的费用是Ci,得到的价值是Wi.求解如何放入背包可以使总价值最大。对于这个问题，每个物品有且只有一件，并且只有放和不放两种状态。F[i,v]表示前i件物品放入一个容量为v的背包中可以获得的最大价值。则其状态转移方程为：

```vim
F[i,v]=max{F[i-1,v],F[i-1,v-Ci]+Wi}
```
该表达式的意思是如果不放第i件物品则转化为“前i-1件物品放入容量我v的背包中”，价值为F[i-1,v];如果放第i件物品，则问题则转化为“前i-1件物品放入v-Ci的背包中”，此时获得的最大的价值为F[i-1,v-Ci]+Wi.
伪代码如下：
```java
F[0,0...v]=0
    for i=1 to N
       for v=ci to V
          F[i,v]=max{F[i-1,v],F[i-1,v-Ci]+Wi}
```
上述的时间和空间复杂度都是O（VN），其中时间复杂度不能再优化，空间复杂度可以优化到O(V)，此处由于篇幅限制，不再说明。
伪代码为：
```java
F [0..V ] ←0
for i ←1toN
  for v ←V toCi
    F[v] ←max{F[v],F[v−Ci]+Wi}

```
&emsp;&emsp;如果不限制每个物品的使用次数，则该问题变成完全背包问题。该问题类似于01背包问题，不同的是每种物品无限个，从物品的角度讲每个物品不再只有取和不取两种状态，而是有取0，1，2，3.。。。[V/Ci]等多种。按照01背包的思路，状态转移方程如下所示：
```vim
F[i,v]=max{F[i-1,v-kCi]+kWi|0<=k<=v}
```
该方程每个状态求解的状态已经不再是常数，我们可以把它转化为01背包问题来解决。其中一个思路是，考虑到第i件物品最多可以取[V/Ci]件，然后可以把第i种物品转化为[V/Ci]件费用及价值均不变的物品，然后求解这个01背包问题。更高效的一种解决方法是将第i件物品才分为费用为Ci2K,价值为Wi2K的若干物品。其中k取遍满足Ci2K<=V的非负整数。其思想是第i件物品总能表示为若干个2K件物品相加。这样一来就将每种物品都拆分成了O(log ⌊V /Ci⌋)件物品。
伪代码如下：
```java
F [0..V ] ←0
for i ←1toN
  for v ←Ci to V
    F[v] ←max{F[v],F[v−Ci]+Wi}
```
该代码和01背包的伪代码只有v的循环次数不同而已。为什么只要循环颠倒就能得到正确的结果呢？让v递减是为了保证F[i,v]是由状态F[i-1,v-ci]递推而来。这正是为了保证每件物品只能选1次。而完全背包每件物品能够选无限次，在考虑加入第i件物品时，需要一个可能已经选入第i种物品的子集F[i,v-Ci]，所以需要v顺序。
&emsp;&emsp;如果第i个物品最多有Mi件可用，则该问题变成了多重背包问题。对于第i件物品有Mi+1种策略，取0，1，2.。M件。则该转移方程为：
```vim
F [i,v] = max{F [i − 1, v − k ∗ Ci] + k ∗ Wi | 0 ≤ k ≤ Mi}
```
如何解这个方程？我们依然考虑二进制的思想，我们把第i件物品转换为若干件物品，使0，1，。。。Mi件的情况均能等价的取若干件等价的商品。方法为：将第i件商品分为若干件01背包中的物品，其中每件物品都有一个系数。这件物品的费用和价值等于原来的费用和价值乘以这个系数。这些系数为1，2，2^2...2^k-1,Mi-2^k+1,且K是满足Mi-2^k+1>0的最大整数。例如Mi为13，则k=3,这个最多取13个的物品被分为系数1，2，4，6的4个物品。
下面为O(logM)时间处理一件多重背包中物品的过程：
```python
def MultiplePack(F,C,W,M) 
	if C · M ≥ V
		CompletePack(F,C,W)
		return
k←1
while k < M
	ZeroOnePack(kC,kW) 
	M ←M − k
	k ← 2k
ZeroOnePack(C · M,W · M)
```

示例部分代码：
```java
package com.yuan.test;


import java.util.Arrays;
import java.util.List;

public class PackUtil {

    public  int N; //物品种类数目
    public  int V; //背包容量
    public  List<Integer> values; //每种物品的代价、价值，此处是特例，相等的
    public  List<Integer> counts; //每种物品的个数
    public  int[] path; //记录最优解路径
    public  int[] numOfEach; //每种多少个，保留结果
    public  int[] transfer; //保留转移数目

    public  int[][] packPath;

    //初始化背包参数
    public  void init(int kind, int volume, List<Integer> valueList, List<Integer> countList) {
        N = kind;
        V = volume;
        values = valueList;
        counts = countList;
    }

    //0-1背包单个物品，F是能得到的总价值，cost是代价，value是单个物品的价值, cur代表转移的时候是单一物品的多少倍，即二进制优化数目
    public  void ZeroOnePack(int[] F, int cost, int value, int cur,int[][] packPath,int index) {
        //System.out.println("cost  "+cost);
        for (int i = V; i >= cost; i--) {
            if(F[i]>=F[i-cost]+value){
                //packPath[index][i]=cost;
            }else {
                packPath[index][i]=packPath[index][i-cost]+cost;
            }
            F[i] = Math.max(F[i], F[i - cost] + value);
            if ((F[i] == F[i - cost] + value) && (F[i - cost] != -1)) {
                path[i] = i - cost;
                transfer[i] = cur;
            }

        }
    }

    //完全背包单个物品，F是能得到的总价值，cost是代价，value是单个物品的价值
    public  void CompletePack(int[] F, int cost, int value,int[][] packPath,int index) {

        for (int i = cost; i <= V; i++) {
            if(F[i]>=F[i-cost]+value){
                //packPath[index][i]=packPath[index][i];
            }else {
                packPath[index][i]=packPath[index][i-cost]+cost;
            }
            F[i] = Math.max(F[i], F[i - cost] + value);
            if ((F[i] == F[i - cost] + value) && (F[i - cost] != -1)) {
                path[i] = i - cost;
                transfer[i] = 1;
            }
        }
    }

    //多重背包单个物品，F是能得到的总价值，cost是代价，value是单个物品的价值，count是物品个数
    public  void MultiplePack(int[] F, int cost, int value, int count,int[][] packPath,int index) {
        if (cost * count >= V) {
            System.out.println("走完全背包");
            //System.out.println(cost);
            CompletePack(F, cost, value,packPath,index);
        } else {
            int k = 1; //二进制优化，把count个物品拆分成0-1背包形式
            while (k < count) {
                //System.out.println(k);
                ZeroOnePack(F, k * cost, k * value, k,packPath,index);
                count -= k;
                k *= 2;
//                System.out.println(Arrays.toString(F));
//                System.out.print("数量 ");
//                System.out.println(Arrays.toString(packPath[index]));
            }
            //System.out.println(count);
            //System.out.println(count);
            ZeroOnePack(F, cost * count, value * count, count,packPath,index);
//            System.out.println(Arrays.toString(F));
//            System.out.print("数量 ");
//            System.out.println(Arrays.toString(packPath[index]));
        }
    }

    //多重背包的极大价值
    public  int MultiplePack(int kind, int volume, List<Integer> valueList, List<Integer> countList) {
        init(kind, volume, valueList, countList);
        int[] F = new int[V + 1]; //不一定有刚好满足的解，所有初始化为0
        Arrays.fill(F, 0);
        F[0] = 0;
        path = new int[V + 1];
        Arrays.fill(path, -1);
        numOfEach = new int[N + 1];
        transfer = new int[V + 1];
        packPath=new int[kind+1][V+1];
        for (int i = 1; i <= N; i++) {
            //注意：价值和数量列表长度都是N+1,第一个是0
            MultiplePack(F, valueList.get(i), valueList.get(i), countList.get(i),packPath,i);
        }
//        for (int i = V; i != 0; i = path[i]) {
//            int cost = (i - path[i]) / transfer[i];
//            for (int j = 1; j <= N; j++) {
//                if (cost == valueList.get(j)) {
//                    numOfEach[j] += transfer[i];
//                }
//            }
//        }
//
//        for (int i = 1; i <= N; i++) {
//            System.out.println("第" + i + "种物品应该取" + numOfEach[i] + "个");
//        }
        int temp=V;
        for(int i=N;i>=1;i--){
            if(temp<0){
                System.out.println("err"+temp);
                break;
            }
            int num=packPath[i][temp];
            temp=temp-num;
            System.out.println("物品编号:"+i+"   占用空间:"+num+"   个数:"+num/valueList.get(i));
        }
        return F[V];
    }

}

```



