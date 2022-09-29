---
layout: post
title: 并查集
date: 2021-11-27 14:11:32 +0800
categories: 数据结构与算法
tags: 并查集 数据结构 算法
author: Rxsi
---

* content
{:toc}

### 算法构成
主要分为两步，find查找 和 union合并
其中在find的过程中可以进行路径的压缩，在union的过程中可以利用size数组统计当前树的节点数，从而实现把节点数少的树合并到节点数多的树
<!--more-->
```cpp
// 初始化
vector<int> father(size);
void build(int size)
{
    for (int i = 0; i < size; ++i)
    {
        father[i] = i; // 初始化时所有节点的父节点都是自身
    }
}

// 查找
int find(int x)
{
    if (father[x] != x)
    {
        father[x] = find(father[x]); // 此处起到了路径压缩的作用
    }
    return father[x];
}

// 合并
vector<int> size(size, 1); // 用以记录树的节点数
void union(int x, int y)
{
    int father_x = find(x);
    int father_y = find(y);
    if (father_x == father_y) return;
    if (size[father_x] > size[father_y])
    {
        father[father_y] = father_x; // 让节点数少的链接到节点数多的
        size[father_x] += size[father_y];
    }
    else
    {
        father[father_x] = father_y;
        size[father_y] += size[father_x];
    }
}
```

### 应用
LeetCode_721：账户合并
```cpp
class Solution {
private:
    vector<int> parent; // 并查集数组，因此需要把邮箱地址转换为int类型的id
    vector<int> size;
    
    void unionParent(int index1, int index2)
    {
        int index1Parent = findParent(index1);
        int index2Parent = findParent(index2);
        if (index1Parent == index2Parent) return;
        if (size[index1Parent] > size[index2Parent])
        {
            parent[index2Parent] = index1Parent; // 节点少的树链接到节点数多的，因为这样会降低树的高度
            size[index1Parent] += size[index2Parent];
        }
        else
        {
            parent[index1Parent] = index2Parent;
            size[index2Parent] += size[index1Parent];
        }
    }
    
    int findParent(int index)
    {
        if (parent[index] != index) parent[index] = findParent(parent[index]); // 不是父节点，此处起到了路径压缩的作用
        return parent[index];
    }
    
public:
    vector<vector<string>> accountsMerge(vector<vector<string>>& accounts) {
        unordered_map<string, int> mailToIndex; // 形成邮箱名 -> 序号的映射
        unordered_map<int, string> IndexToName; // 形成序号 -> 用户名的映射
        int index = 0;
        for (auto& account: accounts)
        {
            string& name =  account[0];
            for (int i = 1; i < account.size(); ++i)
            {
                string& mailName = account[i];
                if (mailToIndex.find(mailName) == mailToIndex.end())
                {
                    IndexToName[index] = name;
                    mailToIndex[mailName] = index++;
                }
            }
        }
        parent.resize(index); // 容量为所有邮箱数量的并查集数组
        size.resize(index, 1); // 用来压缩路径，初始时都为1，代表只有一个节点
        for (int i = 0; i < index; ++i) parent[i] = i; // 初始化父节点为自身
        // 进行并查集的合并操作，这里要遍历原先的accounts，因为IndexToName中的name可能是相同的，但却不是同一个人
        for (auto& account: accounts)
        {
            string& mailName = account[1];
            int index1 = mailToIndex[mailName];
            for (int i = 2; i < account.size(); ++i)
            {
                string& mailName = account[i];
                int index2 = mailToIndex[mailName];
                unionParent(index1, index2); // 关键步骤
            }
        }
        // 完成并查集的合并之后，将相同父节点的所有邮箱名放到同一个列表中
        unordered_map<int, vector<string>> indexToMails;
        for (auto iter = mailToIndex.begin(); iter != mailToIndex.end(); ++iter)
        {
            int parentIndex = findParent(iter->second);
            auto& Vec = indexToMails[parentIndex];
            Vec.emplace_back(iter->first);
        }
        vector<vector<string>> res;
        for (auto iter = indexToMails.begin(); iter != indexToMails.end(); ++iter)
        {
            string& name = IndexToName[iter->first];
            sort(iter->second.begin(), iter->second.end());
            vector<string> res_1{name};
            for (auto iter2 = iter->second.begin(); iter2 != iter->second.end(); ++iter2) res_1.emplace_back(*iter2);
            res.emplace_back(res_1);
        }
        return res;
    }
};
```
