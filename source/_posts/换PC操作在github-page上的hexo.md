---
title: 换PC操作在github page上的hexo
date: 2017-12-25 21:26:56
tags: [hexo , git]
---
&emsp;&emsp;本来是想把工作中的遇到的一些问题也整理到这里做个记录，但是发现hexo在github中的repo是generate后的文件，不能直接```git clone```进行异地编辑，搜了下可以将所有文件```push```到远程其他分支，这倒是个优雅的方法。思路归思路，把步骤记录下。</br>
&emsp;&emsp; 1.进入本地repo路径，我这里是```orxvan.github.io```此时还不是git repo，需要```git init```,然后```git add .```。</br>
&emsp;&emsp; 2. 提交，```git commit -m 'somesthing'```。</br>
&emsp;&emsp; 3. 这里是关键了 ```git remote add origin git@github.com:username/reponame.git```,然后```git push -u origin master```提交。</br>