---
Title: Leetcode 题解 （Mysql）
---

## Mysql

### <a href = "https://www.hackerrank.com/challenges/occupations">Occupations</a>

> 典型的按 某个列的值，进行按列分类查询

+ 这个提的思路是，首先建立一个子查询，将通过Occpution的Index 将 名称加上，即完成对特定的Occpuation进行递增，
+ 然后每个name，和Index 成单一对立的关系。说明现在将其中一个作为 Group选项即可！！在这里明显是要用Index来做Group By聚合
+ 下面解决的问题自然是没有需要一个聚合函数将四个列的属性选出，在这里用了Min函数

+ 第一步 ： 产生一个临时表
	<pre>
	select  
     case 
     when occupation = 'Doctor ' then (@r1 := @r1 +1)  
     when occupation = 'Professor  ' then  @r2 := @r2 +1
     when occupation = 'Singer ' then  @r3 := @r3 +1
     when occupation = 'Actor' then  @r4 := @r4 +1   
     end num,
     case when occupation = 'Doctor ' then name end Doctor,
     case when occupation = 'Professor  ' then name end Professor,
     case when occupation = 'Singer ' then name end Singer,
     case when occupation = 'Actor' then name  end Actor
     from occupations  temp
	</pre> 
 
 这一步完成创建一个 表，将 属性分类，但是会有很多个 NULL 在里面！。

+ 上面的l临时表暂且命名为 temp,利用r1.num 作为Group 对象，但是现在那四个列没有作为select的选项，因为没有聚合参加。所以现在要有一个聚合，这里用了 min，作为聚合函数！！
  <pre>
 select  min(Doctor),min(Professor),min(Singer),min(Actor) from  temp  group by r1.num; 
</pre>


### <a href="https://www.hackerrank.com/challenges/the-report">The Report</a>

> 一个联表查询，不同的是。它的对应关系比较特别，一个表的数据需要对应另一个表的区间值。这个可以直接用一个 联合的判断语句来进行筛选，在这里就时 `on student.mark >= gtades.min_mark && students.mark <= grade.max_mark`

+ **union 和union all 的区别是 union all 会keep duplicate element ，union 会将 重复的元素 移除掉**
+ **没有 所谓的 right inner join 或者 left inner join,只有 inner join，inner join 会 select 出连个表同时m满足的 元素，right join 和 left join会 select 出 右表 或者 左表 全部的元素，和 左表或者右表 满足 join条件的元素，如果 不够 则是 补 null。。。。。**
+ **union 会破坏啷个表的排序！ 所以对两个已经排序的表，在union后还要保持排序状态是不可能的**

### <a href = "https://www.hackerrank.com/challenges/full-score">Top Competitors </a>

> 本题的难度在于怎么解决 某个出现在表中的元素出现的次数，并利用这个次数哦作为筛选条件？

+ 对于 次数作为筛选条件，可以用在排序之后，做一个 case 判断
	<pre>
	 set @pre_id = -1,@count = 0;
	 case 
          when @pre_id = h_id then @count := @count +1  
          else @count := 0
          end count , @pre_id,@pre_id := h_id 
	</pre>
以上利用一个列在作为。元素出现的次数的判断。然后再做一次 select 查询。利用 max(...)作为 aggrate 的条件即可。

+ **注意！ 在 select 语句中的order by 是在 执行完 select 语句，也就是说，是在**

<pre>
set @pre_id = -1,@count = 0;
select  hacker_id,name   from hackers inner join  (select h_id,case 
          when @pre_id = h_id then @count := @count +1  
          else @count := 0
          end count , @pre_id,@pre_id := h_id from difficulty inner join  (select 
          submissions.hacker_id h_id,submissions.score score,challenges.difficulty_level d_level from
          submissions inner join challenges on submissions.challenge_id = challenges.challenge_id 
          order by h_id)r1 on r1.d_level = difficulty_level && r1.score = difficulty.score) r2 on 
	     hackers.hacker_id = r2.h_id  where r2.count >= 1 group by hacker_id order by max(count) 
    	desc ,hacker_id;
 
</pre>

**在利用 一个元素的max（xxx） ，或者 case 时，没有必要将这个列作为 select 的选项之一 select 出来！**