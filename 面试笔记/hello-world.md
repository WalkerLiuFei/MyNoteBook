---
title: 搭建Github page笔记遇到的坑
---
## 配置Hexo遇到的坑
1. 在配置hexo的_config.yml文件时
	<pre>
	deploy:
	  type: git //前面要加空格！
	  repo: git@github.com:WalkerLiuFei/WalkerLiuFei.github.io.git //这里不要用https指向的地址
	  branch: master
	</pre>
2. 每次提交新文章的时候，hexo会清空github仓库（坑！），所以我在其他地方提交的文件，例如定向文件CNAME会被清空掉，所以这个文件药房你的hexo博客目录下面的source文件夹下！
