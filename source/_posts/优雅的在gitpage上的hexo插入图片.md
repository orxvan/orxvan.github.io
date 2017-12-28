---
title: 优雅的在github page上的hexo插入图片
date: 2017-12-28 22:53:40
tags:
- hexo
---
&emsp;&emsp;会者不难难者不会，使用图床插入图片的问题在于未来的不确定性，而且看起来很ugly。零零散散找了半天，步骤很简单：
- _config.yml 中更改post_asset_folder:true;这个很多博客都说了，但是会有问题，首页不显示图片。
- ```npm install https://github.com/CodeFalling/hexo-asset-image --save```这个点很多没提，是解决是否完美显示的关键。
- 现在```hexo new "new post"```就有个同名文件夹了，图片放进去，md语法使用相对路径就行了? </br>

&emsp;&emsp;不过和图床方法相比的在MarkdownPad中写这个方法看不到预览图。还是没有那么完美。
