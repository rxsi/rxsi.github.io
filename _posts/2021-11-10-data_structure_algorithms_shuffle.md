---
layout: post
title: 洗牌算法
date: 2021-11-10 10:22:43 +0800
categories: 数据结构与算法
tags: Fisher-Yates 蓄水池采样 数据结构 算法
author: Rxsi
---

* content
{:toc}

### Fisher–Yates shuffle
每次从后往前等概率抽取一个下标 index(0~i)，然后交换该 index 与下标 i 的值

时间复杂度：O(n)

空间复杂度：O(1)

缺点：会打乱原数组，且是从后往前计算，因此不适用于动态变化的数组

<!--more-->
证明：设目标元素为 i

1. 第一次才抽中i的概率为：   1/n
2. 第二次才抽中i的概率为：   (n-1)/n * 1/(n-1) = 1/n
3. 第N次才抽中i的概率为：    (n-1)/n * (n-2)/(n-1) * (n-3)/(n-2)... * 1/1 = 1/n

从上面的证明可知，对于任意的元素在任意次数被抽中的概率都是 1/n
Python版本：
```python
import random
def shuffle(nums):
    for i in range(len(nums)-1, -1, -1):
        j = random.randint(0, i) # [0, i]范围
        nums[i], nums[j] = nums[j], nums[i]
```

C++版本：
```cpp
#include <vector>
#include <iostream>
#include <random>

using namespace std;

void shuffle(vector<int>& nums)
{
    int n = nums.size();
    for (int i = n-1; i >= 0; --i)
    {
        int j = rand() % (i + 1); // 注意rand()是伪随机，因此程序重新运行得到的随机结果都是一样的，但是在一次程序内连续调用是不同的，这个在实际应用时要注意，可以使用srand()设置随机种子
        swap(nums[i], nums[j]); 
    }
}

int main()
{
    vector<int> nums{0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10};
    shuffle(nums);
    for (auto& num: nums) cout << num << " ";
    cout << endl;
}
/*
输出：5 0 1 2 6 10 3 4 9 7 8 
*/
```

### 蓄水池抽样

**特点：如果数据集合的量特别大或者还在增长（相当于未知数据集合总量），该算法依然可以等概率抽样**

证明：设目标抽取数为m，而 i 为目标数，分两种情况讨论:

1. i为在<=m 时进入的池子，且最后留在池子的概率：

    当池子未满时，进入池子的概率为 1；
    当池子满，以 m+1 为例，加入了新值需要随机淘汰一个，那么 i 值没有被淘汰的概率为 m/(m+1)；
    以 m+2 为例，此时池子里面只有 m 个值，但是为了保证对于每一个加入值都有公平的概率，此时要考虑之前已经加入过的值，因此池子总数要当成 m+2，因此 i 值没有被淘汰的概率为 (m+1)/(m+2)
    以此类推，n次的时候概率为 (n-1)/n，因此最后的总概率为：

    ![litter_than_m.png](/images/data_structure_algorithms_shuffle/litter_than_m.png)

2. i为在 >m 时进入的池子，且最后留在池子的概率：

    如果 i = m+1，那么能够成功进入池子的概率为 m/(m+1)，即 m/i；
    后面每次不被替换的概率为：1-1/(i+1), 1-1/(i+2) .... 1-(1/n)，因此最后的总概率为：

    ![better_than_m.png](/images/data_structure_algorithms_shuffle/better_than_m.png)

Python版本：
```python
import random
def pool_shuffle(nums):
    M = 10 # 假设只取样10个
    res = []
    count_times = 0
    for num in nums:
        count_times += 1
        if count_times > M:
            rand_index = random.randint(0, count_times)
            if rand_index <= M:
                res[rand_index-1] = num
        else:
            res.append(num)
    return res

nums = []
for i in range(100000):
    nums.append(i)
print(pool_shuffle(nums))
```

C++版本：
```cpp
#include <vector>
#include <iostream>
#include <random>

using namespace std;

void shuffle(vector<int>& nums, vector<int>& res, int n) // n是要抽取的数据量
{
    int countTimes = 0;

    for (auto num: nums)
    {
        ++countTimes;
        if (countTimes <= n) res.push_back(num);
        else
        {
            int j = rand() % (countTimes + 1);
            if (j <= n) res[j-1] = num; // 随机到了前n个位置，那么可以换入
        }
    }
}

int main()
{
    vector<int> nums{0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10};
    vector<int> res;
    shuffle(nums, res, 5);
    for (auto& num: res) cout << num << " ";
    cout << endl;
}
/*
输出：0 1 6 10 4
*/
```