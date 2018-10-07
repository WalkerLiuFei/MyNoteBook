---
title: 字典树
---

# 字典树(trie)

----

> 在leetcode 上刷题遇到的一种数据结构，以前没听说过，怪我孤陋寡闻，经过了解发现应用还是很广的。觉得有必要记录下
<a href =  "https://leetcode.com/problems/implement-trie-prefix-tree/"> leetcode 208</a>---
<a href = "https://zh.wikipedia.org/wiki/Trie">trie的维基百科</a>
 
+ 概念
 + 根节点不包含字符串 
 + 从根节点到某一叶子节点，路径上组成的所有字符就是该节点对应的字符串
 + 每个节点的公共前缀作为一个字符节点保存 
+ 应用
 + 词频统计：比hash或者堆要节省空间，
 + 前缀匹配：前缀匹配的算法复杂度是O(1)， 要查找匹配前缀字符串的长度
 
## 实现

```Java

class TrieNode{
    TrieNode[] childs = new TrieNode[26]; // a-z
    int count; //frequence;
    char prefix; //当前节点的字符前缀
    public TrieNode(char prefix){
        this.prefix = prefix; 
        count = 1; //root 的count 初始化为0，所以要
    }
    public TrieNode(){}
    public void addCount(){
        this.count++;
    }
    public void setPrefix(char prefix){
        this.prefix = prefix;
    }
}

public class Trie {
    private TrieNode root;

    public Trie() {
         root = new TrieNode();
    }

    public void insert(String word) {
        TrieNode searchNode = root;
        for (int level = 0;level < word.length();level++){
            char currentChar = word.charAt(level);
            TrieNode node = searchNode.childs[currentChar-'a'];
            if (searchNode.childs[currentChar-'a'] == null)      //如果这个前缀的还是null，那么新建插入一个
                searchNode.childs[currentChar-'a'] = new TrieNode(currentChar);
            else
                node.count++;
            searchNode = searchNode.childs[currentChar-'a'];
        }
    }

    public boolean search(String word) {
        TrieNode searchNode = root;
        for (int level = 0;level < word.length();level++){
            char currentChar = word.charAt(level);
            TrieNode node = searchNode.childs[currentChar-'a'];
            if (node == null)
                return false;
            searchNode = node;
        }
        int childCount = 0;
        for (TrieNode trieNode : searchNode.childs){
            if (trieNode == null)
                continue;
            childCount += trieNode.count;
        }
        return searchNode.count > childCount;
    }


    public boolean startsWith(String prefix) {
        TrieNode searchNode = root;
        for (int level = 0;level < prefix.length();level++){
            char currentChar = prefix.charAt(level);
            TrieNode node = searchNode.childs[currentChar-'a'];
            if (node == null)
                return false;
            searchNode = node;
        }
        return true;
    }
}

```