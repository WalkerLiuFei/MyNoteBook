---
title: 最近leetcode刷题总结（16/11/29）
---

# backtracking (回溯算法)

---
 
回溯算法属于暴力解法。一般配合 DFS或者BFS使用，通过适当的剪枝来实现降低时间复杂度

## <a href = "https://leetcode.com/problems/generate-parentheses/"> Generate Parentheses</a>

```Java
     List<String> result = new ArrayList<>();
    public List<String> generateParenthesis(int n) {
        backTracking(2*n,n,1,0,1,"("); //长度为2*nde 字符串满足条件！
        return result;
        /*
          思路：
          1:利用indicateFlag 标示是否添加 右括号。左括号添加一次加一，右括号添加一次减一。不能为 负数！
          2:构建一个char[2n-1] 的状态空间。每次都去尝试左括号和右括号。不满足的话就回到上一次的状态。
            其实想层次遍历二叉树一样。利用BFS！队列
         3 : 假设左子 为右半括号，右子为左半括号
         */
    }

    private void backTracking(int resLength,int paraNum, int leftAdded,int rightAdded,int flag, String cont) {
        if (flag < 0 || rightAdded > paraNum || leftAdded > paraNum) //这里就是实现的剪枝，不满足条件的，就不继续执行下去
            return;
        if (cont.length() == resLength){
            result.add(cont);
        }
        if (flag > 0)  
            backTracking(resLength,paraNum,leftAdded,rightAdded+1,flag-1,cont+")");

        if (flag >=0)
            backTracking(resLength,paraNum,leftAdded+1,rightAdded,flag+1,cont+"(");
    }
```

## <a href= "https://leetcode.com/problems/combination-sum/">Combination Sum</a>

+ 这个题解和其他Combination Sum的题目都很相似，利用DFS，经过适当剪枝，将满足条件的结果加入集合。。。。

```Java
    public List<List<Integer>> combinationSum(int[] candidates, int target) {
        if (candidates.length < 1)
            return null;
        List<List<Integer>> result = new ArrayList<>();

       for (int index = 0; index < candidates.length; index++){  //利用一个外部循环可以减少递归的次数
            List<Integer> list = new ArrayList<>();
            list.add(candidates[index]);
            recursive(result, list, candidates, target,index,candidates[index]);
       }

        return result;
    }

    private void recursive(List<List<Integer>> result, List<Integer> list, int[] candidates, int target, int restrict,int currentValue) {
        if (currentValue == target){
            result.add(list);
            return;
        }

       for(int index2 = restrict;index2 < candidates.length;index2++){
            int value = currentValue + candidates[index2];
        
            if (value <= target){
                list.add(candidates[index2]); // add the element
                ArrayList newList = new ArrayList();
                newList.addAll(list);
                recursive(result, newList, candidates, target,index2,value);
                list.remove(list.size() - 1); // remove the top element
            }
            
        }
    }
```


## <a href = "https://leetcode.com/problems/restore-ip-addresses/">Restore IP Addresses</a>

+ 这个题解最需要注意的是IP地址后面的3个节点不能以0开头，一开始做题就卡到这里了

```Java
   public List<String> restoreIpAddresses(String s) {
        List<String> result = new ArrayList<>();
        StringBuilder builder = new StringBuilder();
        dfs(s, result, builder, 0);
        return result;
    }

    private void dfs(String restString, List<String> result, StringBuilder builder, int currentLength) {
        if (restString.length() + currentLength < 4) //剩余字符串的长度不足以形成一个有效的IP 地址
            return;
        if (currentLength == 3 
                    && (restString.length() < 4 
                            && Integer.parseInt(restString) <= 255)) {
            if (restString.length() >1 && restString.startsWith("0"))
                return;
            StringBuilder stringBuilder = new StringBuilder(builder);
            String value = stringBuilder.append(restString).toString();
            result.add(value);
            return;
        } else if (currentLength >= 3) //不满的情况，>= 3，大于 255 ||
            return;
        int limit = Math.min(3, restString.length());
        for (int index = 1; index <= limit; index++) {
            String sub = restString.substring(0, index);
            if (sub.startsWith("0") && sub.length() > 1)
                continue;
            if (index == 3 && (sub.length() > 3 || Integer.parseInt(sub) > 255))
                continue;
            builder.append(sub);
            builder.append(".");
            dfs(restString.substring(index, restString.length()), result, builder, currentLength + 1);    //剪切第0个字符
            builder.delete(builder.length() - index - 1, builder.length());
        }
    }
``` 
