---
layout: post
title: 最短路径
date: 2021-12-10 14:41:00 +0800
categories: 数据结构与算法
tags: Dijkstra Floyd Bellman-Ford spfa 数据结构 算法
author: Rxsi
---

* content
{:toc}

### 多源最短路径
#### Floyd
统计 graph 图中**任意两点**之间的最短距离

动态规划思想：i，j 两点之间的最短距离，通过枚举中间点 k ，计算其最小值

动态方程：dp[i][j] = min(dp[i][j], dp[i][k]+dp[k][j])

时间复杂度：O(n^3)

空间复杂度：O(n^2)

最终得到的结果是所有点之间的最短距离，可以输出路径，但是效率不高，不过这种算法可以有负权路，但是不能有负权环。
<!--more-->

Python版本：
```python
import copy
import time

def floyd(graph):
    dist = copy.deepcopy(graph)
    for k in range(len(graph)):
        for i in range(len(graph)):
            if dist[i][k] == INF: # 剪枝
                continue
            for j in range(len(graph)):
                if dist[k][j] == INF: # 剪枝
                    continue
                dist[i][j] = min(dist[i][j], dist[i][k] + dist[k][j]) 
    return dist
                
INF = float('inf')
graph = [[0, 5, INF, 10],
        [INF, 0, 3, INF],
        [INF, INF, 0, 1],
        [INF, INF, INF, 0]
        ]
floyd(graph)
```

C++版本：
```cpp
#include <iostream>
#include <vector>
using namespace std;

#define M INT_MAX/2

/*
该算法可以计算有负权的边，因为他对于每一组点的计算都是通过遍历所有点之后才确定的，因此有负权的边最终也会被正确的计算
但是不可以计算有负权环的图，因为每次都走负权环，则每次的路径都在减少，最终是无穷小
*/

void floydFinder(vector<vector<int>>& dist, vector<vector<int>>& path)
{
    int n = dist.size();
    vector<vector<int>> dp(dist);
    for (int k = 0; k < n; ++k)
    {
        for (int i = 0; i < n; ++i)
        {
            for (int j = 0; j < n; ++j)
            {
                if (dp[i][j] > dp[i][k] + dp[k][j])
                {
                    dp[i][j] = dp[i][k] + dp[k][j];
                    path[i][j] = k;
                }
            }   
        }
    }
    return;
}

void findPath(vector<vector<int>>& path, int start, int end, vector<int>& route)
{
    if (path[start][end] == -1) // 两点不通
    {
        if (route.empty())
        {
            route.push_back(start);
            route.push_back(end);
        }
        else
        {
            route.push_back(end);
        }
        return;
    }
    else
    {
        int k = path[start][end];
        findPath(path, start, k, route);
        findPath(path, k, end, route);
        return;
    }
}


int main()
{
    vector<vector<int>> dist {
        {0, 5, M, 10},
        {M, 0, 3, M},
        {M, M, 0, 1},
        {M, M, M, 0}
    };
    int n = dist.size();
    vector<vector<int>> path(n, vector<int>(n, -1)); // 存储的是i、j两点之间最短距离需要经过的点
    floydFinder(dist, path);
    vector<int> route;
    findPath(path, 0, 3, route);
    for (auto& i: route)
    {
        cout << i << " ";
    }
    cout << endl;
}
/*
输出：0 1 2 3 
*/
```

### 单源最短路径
#### Dijkstra
统计graph图中**指定点的到其他点**的最短距离

贪心算法思想，通过每次遍历贪心的选择距离 **起点** 最短距离的点作为指定点，然后确定其到其他点的最短距离。

时间复杂度：O(n^2 + E) E是每次确定最短距离点的时间复杂度，可以使用小顶堆优化，这里使用的是直接遍历，即O(n)复杂度，共遍历n次，因此 E=O(n^2)

空间复杂度：O(n)

最终得到的结果是目标点到其余各点的最短距离，而不能输出路径，但是不能计算负权图，更不能计算负权环的图

Python版本：
```python
INF = float('inf')
def dijkstra(point, graph):
    dist = list.copy(graph[point]) # point对应的点将会是0,dist中保存的是指定点到其他各点的直接距离
    check_dist = [False] * len(graph) # 统计已经确定的点
    check_dist[point] = True
    for i in range(len(graph)-1):
        min_index = findminindex(dist, check_dist) # 每次贪心的选择最短距离点
    if min_index == INF: #此路不通...
        break
    check_dist[min_index] = True
    for v in range(len(graph)): # 以当前最短距离点为起始,更新dist中还未确定且原始指定点到该点的距离(dist[v])大于从原始指定点到当前最短距离点+当前最短距离点到该点的距离(dist[min_index]+graph[min_index][v])
        if check_dist[v] == False and graph[min_index][v] != INF and dist[v] > dist[min_index] + graph[min_index][v]:
            dist[v] = dist[min_index] + graph[min_index][v]
    return dist

def findminindex(dist, check_dist):
    temp_min = INF
    min_index = INF
    for index in range(len(dist)):
        if dist[index] < temp_min and check_dist[index] == False:
            temp_min = dist[index]
            min_index = index
    return min_index
graph = [[0, 5, INF, 10],
        [INF, 0, 3, INF],
        [INF, INF, 0, 1],
        [INF, INF, INF, 0]
        ]
dijkstra(0, graph)


# 使用优先队列优化:
def dijkstra(point, graph):
    dist = []
    for index in range(len(graph[point])):
        dist.append((graph[point][index], index)) # (val, index)，dist存储的是point这个点到其他点的距离
    heapq.heapify(dist) # heapq作为优先队列,将dist构建为小顶堆,dist[0]为最小距离点
    res_list = [INF] * len(graph)
    while dist:
        min_val, min_index = heapq.heappop(dist)
        res_list[min_index] = min_val
        if min_val == INF:  # 最小值已经为INF
            break
        for dist_index, (val, index) in enumerate(dist):
            if graph[min_index][index] != INF and val > min_val + graph[min_index][index]: # 已经确定了一个点，那么遍历这个点到其他的点的距离是否更短
                dist[dist_index] = (min_val + graph[min_index][index], index)
        heapq.heapify(dist)
    return res_list
```

C++版本 - 邻接矩阵：
```cpp
#include <iostream>
#include <vector>
#include <queue>
using namespace std;

/*
如果使用Python实现将会更加容易，因为Python可以直接更新优先队列中的数，然后再调整堆即可
该算法是无法处理负权边的，因为我们每次选点都是基于：当前未找到的最短路的点中，距离源节点最近的点都是已经无法继续松弛的了
而如果存在负权边，比如[[0, 2, 3], [M, 0, M], [M, -2, 0]]，如果根据dijkstra的规则，首先计算的点将会是1，而其实0 -> 2 -> 1才是真正最短的距离
当前算法自然也就不可以处理负权环的问题了。

根据图的类型，分为两种方式：
1. 如果图是邻接矩阵(本题就是邻接矩阵)，一般是稠密图，也就是计算源节点到其他节点的距离都需要遍历，因此直接使用priority_queue优先队列是没有什么意义的，都是O(n2)，如果是Python那种就有意义。当然如果C++使用heap算法自己去构建和Python那种优先队列一样，则就可以提升一定的性能
2.如果图是邻接表，一般是稀疏图，因此使用优先队列就可以优化，因为我们可以根据点直接拿到他能到达的所有点，一般这么定义：vector<vector<pair<int, int>>> path; path里面包含的就是key能够到达的所有点的列表
见https://leetcode.cn/problems/network-delay-time/

*/

#define M INT_MAX/2


vector<int> dijkstraFinder(int point, vector<vector<int>>& dist, vector<int>& result)
{
    priority_queue<pair<int, int>, vector<pair<int, int>>, greater<pair<int, int>>> Priority_Queue;
    vector<bool> visited(dist.size(), false);
    Priority_Queue.emplace(0, point);
    while (!Priority_Queue.empty())
    {
        auto p = Priority_Queue.top();
        Priority_Queue.pop();
        if (p.first > result[p.second]) continue; // 因为这种做法可能会加入重复的点，因此需要忽略那些不合适的值
        result[p.second] = p.first;
        visited[p.second] = true;
        for (int i = 0; i < dist.size(); ++i)
        {
            if (visited[i]) continue;
            if (dist[p.second][i] + p.first <= dist[point][i])
            {
                Priority_Queue.emplace(dist[p.second][i] + p.first, i);
            }
        }
    }
    return;
}

int main()
{
    vector<vector<int>> dist {
        {0, 5, M, 10},
        {M, 0, 3, M},
        {M, M, 0, 1},
        {M, M, M, 0}
    };
    vector<int> result(dist.size(), M); // 先初始化到所有点的距离都为M
    dijkstraFinder(0, dist, result);
    for (auto& num: result) cout << num << " ";
    cout << endl;
} 

/*
输出：0 5 8 9 
*/
```
C++版本 - 邻接表：
```cpp
class Solution {
public:
    int networkDelayTime(vector<vector<int>>& times, int n, int k) {
        // 邻接表方式
        int M = INT_MAX / 2;
        vector<vector<pair<int, int>>> path(n);
        for (auto& v: times) path[v[0]-1].emplace_back(v[1]-1, v[2]);
        priority_queue<pair<int, int>, vector<pair<int, int>>, greater<pair<int, int>>> myQue;
        myQue.emplace(0, k-1);
        vector<int> dist(n, M);
        while (!myQue.empty())
        {
            auto p = myQue.top();
            myQue.pop();
            if (dist[p.second] < p.first) continue;
            dist[p.second] = p.first;
            for (auto& e: path[p.second])
            {
                int len = e.second + dist[p.second];
                if (len < dist[e.first])
                {
                    dist[e.first] = len;
                    myQue.emplace(len, e.first); 
                }
            }
        }
        int ret = 0;
        for (auto& num: dist) ret = max(ret, num);
        return ret == M ? -1 : ret;
    }
};
```

#### Bellman-Ford
算法的思想很简单就是直接遍历所有可能的点进行松弛操作，假设有N个点，E条边，则最终算法的时间复杂度为O(NE)，效率不高，但是可以处理负权边
要判断负权环，只需要在上面遍历完成之后更遍历一次，如果有任意一条边发生了更新，则说明有负权环。

```cpp
#include <iostream>
#include <vector>
#include <limits.h>
using namespace std;

/*
单源最短路径算法
相较于dijkstra算法，这个算法可以包含负权边和负权环
在伪代码中返回的是true/false，即用来检测是否有环
时间复杂度是O(V*E)，V是顶点数，E是边的数量，这个是在当为领接表时才有的性质

1. 创建源顶点 v 到图中所有顶点的距离的集合 distSet，为图中的所有顶点指定一个距离值，初始均为 Infinite，源顶点距离为 0；
2. 计算最短路径，执行 V - 1 次遍历；
        对于图中的每条边：如果起点 u 的距离 d 加上边的权值 w 小于终点 v 的距离 d，则更新终点 v 的距离值 d；

3. 检测图中是否有负权边形成了环，遍历图中的所有边，计算 u 至 v 的距离，如果对于 v 存在更小的距离，则说明存在环；
*/

#define M INT_MAX/2

vector<int> result;

bool bellmanFordFinder(int point, vector<vector<int>>& dist) // 这里用的是邻接矩阵
{
    int n = dist.size();
    result = vector<int>(n, M);
    for (int i = 0; i < n; ++i) result[i] = dist[point][i]; // 初始距离
    for (int i = 0; i < n; ++i)
    {
        for (int j = 0; j < n; ++j)
        {
            if (dist[i][j] + result[i] < result[j])
            {
                result[j] = dist[i][j] + result[i];
            }
        }
    }
    // 松弛过一次之后，如果还能松弛，那么就代表有环
    for (int i = 0 ; i < n; ++i) 
    {
        for (int j = 0; j < n; ++j)
        {
            if (result[j] > dist[i][j] + result[i]) return false;
        }
    }
    return true;
};

int main()
{
    vector<vector<int>> dist {
        {0, -5, M, 10},
        {-3, 0, 3, M},
        {M, M, 0, 1},
        {M, M, M, 0}
    };
    if (bellmanFordFinder(0, dist))
    {
        for (auto& num: result) cout << num << " ";    
    }
    else
    {
        cout << "have ring";
    }
    cout << endl;
}
/*
输出：have ring
*/
```

#### spfa
在 bellman_ford 算法中，每次都需要遍历所有的边，但是实际上只有在上次的relax中发生了变化的边才会导致其他边发生变化，
因此可以使用队列（先进先出）记录上次变化的边，这样就可以提升速度了。
```cpp
#include <iostream>
#include <queue>
#include <vector>
#include <limits.h>
using namespace std;

#define M INT_MAX/2

bool spfa(int point, vector<vector<int>>& dist, vector<int>& result)
{
    int n = dist.size();
    // for (int i = 0; i < dist.size(); ++i) result[i] = dist[point][i]; // 因为使用了队列进行记录，因此这里不需要这样做，只需要标记point即可
    result[point] = 0;
    queue<int> Que;
    Que.push(point);
    vector<int> countTimes(n, 0);
    countTimes[point]++;
    while (!Que.empty())
    {
        int p = Que.front();
        Que.pop();
        for (int i = 0; i < n; ++i)
        {
            if (result[p] + dist[p][i] < result[i]) // result[p]代表的是point->p的最短距离; dist[p][i]代表的是p->i的当前距离; result[i]代表的是p->i的最短距离
            {
                result[i] = result[p] + dist[p][i];
                countTimes[i]++;
                if (countTimes[i] > n) return false; // 如果存在负权环，则会一直进入队列，因此要统计进入队列的次数，超过总点数则代表存在负权环
                Que.push(i);
            }
        }
    }
    return true;
}

int main()
{
    vector<vector<int>> dist {
        {0, 5, M, 10},
        {M, 0, 3, M},
        {M, M, 0, 1},
        {M, M, M, 0}
    };
    vector<int> result(dist.size(), M);
    if (spfa(0, dist, result))
    {
        for (auto& num: result) cout << num << " ";    
    }
    else
    {
        cout << "have ring";
    }
    cout << endl;
} 
/*
输出：0 5 8 9 
*/
```