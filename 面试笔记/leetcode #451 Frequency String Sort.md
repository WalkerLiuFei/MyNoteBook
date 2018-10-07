---
title: Solution of Leetcode #451
---

# Solution of leetcode #451

> 自从开始刷leetcode以来最高的排位，84%的beat率，更重要的是从构思到实现加debug只用了20分钟！感觉有必要记录下！

下面是具体实现！

```Java 
  public String frequencySort(String s) {
        if (null == s || 0 == s.length() )
            return s;
        char[] array = s.toCharArray();

        Arrays.sort(array);

        List<StringBuilder> elements = new ArrayList<StringBuilder>();
        elements.add(new StringBuilder().append(array[0]));
        for (int index = 1;index < array.length;index++){
            char c = array[index];
            StringBuilder strBuilder = elements.get(0);
            if (strBuilder.charAt(0) == c)
                strBuilder.append(c);
            else
                elements.add(0,new StringBuilder().append(c));
        }

        Collections.sort(elements, new Comparator<StringBuilder>() {
            public int compare(StringBuilder o1, StringBuilder o2) {
                return o2.length() - o1.length();
            }
        });

        StringBuilder stringBuilder = new StringBuilder();
        for (StringBuilder element : elements)
            stringBuilder.append(element);

        return stringBuilder.toString();
    }

```

leetcode 上的runtime 最后是23ms！算法思想很简单，就是先排序。然后利用排序好的字符串进行divide形成list，排序号list里面的元素后进行重装即可。很简单...

# 总结：

算法实现实现起来其实蛮简单，不过重要的是理解java底层的排序算法实现。这正是下一步要做的工作！ 