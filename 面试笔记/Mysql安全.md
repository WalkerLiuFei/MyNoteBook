---
title: Mysql安全
---

# Sql 注入

+ 利用绑定变量，
+ 利用正则表达式对用户输入的信息进行过滤筛选

# <a href = "https://dev.mysql.com/doc/refman/5.7/en/sql-mode.html">Sql Mode</a>

+ 根据不同的业务需求来选择 设置不同 sql_mode
+ Sql mode 可以解决的问题
 + 通过设置SQL Mode,可以完成不同严格程度的数据校验，有效的保证数据准确性
 + 通过设置SQL Mode为ANSI模式，来保证大多数的SQL符合标准的SQL语法，这样来保证在不同的Database之间切换时不需要对SQL进行大量的修改
 + 通过设置sql mode，使得Mysql上的数据更方便迁移到目标数据库中
+ 利用`SET GLOABLE/SESSION SQL_MODE = ‘modes’` 来对SQL的的模式进行设定
+ 最常见的几种MODE 
 + ANSI: 这种模式使得SQL语句必须符合标准sql语法 --->ANSI REAL_AS_FLOAT, PIPES_AS_CONCAT, ANSI_QUOTES, IGNORE_SPACE, and (as of MySQL 5.7.5) ONLY_FULL_GROUP_BY.
 + STRICT_TRANS_TABLES :
 + TRADITIONAL: 这个模式可以用一句话概述： give an error instead of a warning.**插入和更新操作会在错误出现时立刻中止，如果你不是在用一个事务性的存储引擎的话。这可能不是你想要的结果，因为被中止的操作并不会被回滚**
+ Mode 分类
 <font size = 3>
 + ALLOW_INVALID_DATES
 + ANSI_QUOTES
 + ERROR_FOR_DIVISION_BY_ZERO
 + HIGH_NOT_PRECEDENCE
 + IGNORE_SPACE	
 + NO_AUTO_CREATE_USER
 + NO_AUTO_VALUE_ON_ZERO
 + NO_BACKSLASH_ESCAPES
 + NO_DIR_IN_CREATE
 + NO_ENGINE_SUBSTITUTION
 + NO_FIELD_OPTIONS
 + NO_KEY_OPTIONS
 + NO_TABLE_OPTIONS
 + NO_UNSIGNED_SUBTRACTION
 + NO_ZERO_DATE
 + NO_ZERO_IN_DATE
 + ONLY_FULL_GROUP_BY : 指的意思就是 所要query的列 必须有唯一性，被aggrate 函数修饰。或者是所选列的是primary key
 + PAD_CHAR_TO_FULL_LENGTH
 + PIPES_AS_CONCAT : 将 '||'视为 字符串连接符，
 + REAL_AS_FLOAT
 + STRICT_ALL_TABLES
 + STRICT_TRANS_TABLES
</font>
+ Sql Mode 常见功能
 + 校验日期数据的合法性
 + 严格模式：

0	25	16:05:05	create table t (d datetime)	Error Code: 1399. XAER_RMFAIL: The command cannot be executed when global transaction is in the  ACTIVE state	0.000 sec