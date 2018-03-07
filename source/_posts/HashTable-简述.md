---
title: HashTable 简述
tags:
    - hashtable
categories:
    - 算法
date: 2016-06-09 00:47:39
---

### 解释
使用了映射函数，把值映射到对应的位置，key－> address， address是表中的存储位置，不是实际的地址；
 
### Hash 函数设计
分布合理，冲突少，利用率平衡，利用率高了，冲突多，利用率低了，冲突小点；
(1)直接定址发：
address(key) = key -2000
(2)平方取中法， 对关键字进行平方运算，然后取中间的几个数字做为hash地址
address(key) =str( math.pow(key, 2) )[2, -2]
(3)折叠法， 关键的字做一个运算
address(key) = key[0] + key[1] + key[2]
(4)除留取余
知道hash表带最大长度m， 取不大于m的质数p，对关键字进行取余运算
address(key) = key%p
p的选取很重要，因为p选择的好，可以最大限度降低冲突，p一般选择小于m的最大质数
 
### Hash表的大小选择；
(1)根据存储的数量和分布来选择；
(2)动态分配，这时候需要重新计算hash地址；redis 中就有这种模式
 
### 冲突解决；
(1)开发定址法：冲突后按照一定的策略需要寻找空位
(2)链地址法：链地址法
 
 
### hashtable 的桶数选择质数的原因
该段摘自http://blog.csdn.net/liuqiyao_01/article/details/14475159
设有一个哈希函数
H( c ) = c % N;
当N取一个合数时，最简单的例子是取2^n，比如说取2^3=8,这时候
H( 11100(二进制） ) = H( 28 ) = 4
H( 10100(二进制) ) = H( 20 ）= 4
这时候c的二进制第4位（从右向左数）就”失效”了，也就是说，无论第c的4位取什么值，都会导致H( c )的值一样．这时候c的第四位就根本不参与H( c )的运算，这样H( c )就无法完整地反映c的特性，增大了导致冲突的几率．
取其他合数时，都会不同程度的导致c的某些位”失效”，从而在一些常见应用中导致冲突．
但是取质数，基本可以保证c的每一位都参与H( c )的运算，从而在常见应用中减小冲突几率．
 
 
 
下面是自己实现的一个简单的hashTable， 很多问题没有考虑进去，也没有编译测试，仅作为理解
如果想看C++ STL源码，可以参考
https://gcc.gnu.org/onlinedocs/libstdc++/libstdc++-html-USERS-4.1/hashtable-source.html
http://blog.csdn.net/KangRoger/article/details/38640943
```
 
#include <stdio.h>
 
#define M 100
#define Hash_key = 29
 
template <class T>
struct HashNode
{
    T data;
    int key;
    int isNull;
    HashNode<T>* pNext;
 
    HashNode(){
        pNext = NULL;
        isNull = 1;
    }
}; 
 
 
template <class T>
class HashTable
{
private:
    HashNode<T> m_HashNodes[M];
 
    int getHashAddress(int key) {
        return key % Hash_key;
    }
 
 
public:
    HashTable(){
        // for (i=0; i<M; i++) {
        //     m_HashNodes[i].isNull = 0;    
        // }
    }
 
 
    bool insert(int key,T data) {
        int address = this.getHashAddress(key);
        if (m_HashNodes[address].isNull == 1) {
            m_HashNodes[address].data = data;
            m_HashNodes[address].isNull == 0;
        }
        else {
            // while (m_HashNodes[address].isNull == 0 && address <M) { // 线性探测法， 开发地址法
            //     address++；
            // }
            // if (address == M) {
            //     return false;
            // }
            // 
            HashNode<T>* pTmpNode = m_HashNodes[address].pNext; // 链地址法
            HashNode<T>* pCurNode = NULL
            while (pTmpNode != NULL) {
                pCurNode = pTmpNode;
                pTmpNode = pTmpNode->pNext;
            }
            pCurNode->pNext = new HashNode<T>();
            pCurNode->pNext->data = data;
            pCurNode->pNext->key = key;
            pCurNode->pNext->isNull = 0;
        }
    }
 
    HashNode<T> find(int key) {
        int address = this.getHashAddress(key);
        HashNode<T> node = m_HashNodes[address];
        if (node.key == key){
            return &node.data;
        }
        else {
            pCur = m_HashNodes[address].pNext;
            while (pCur != NULL){
                if (pCur->key == key) {
                    return pCur;
                }
                else {
                    pCur = pCur->pNext;
                }    
            }
            return NULL
        }
    }
 
}
```