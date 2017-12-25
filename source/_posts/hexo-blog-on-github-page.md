---
title: hexo blog on github page
date: 2017-12-24 13:29:13
tags: config
---
&emsp;&emsp;主要方法来自于[http://www.lovebxm.com/2017/05/30/buildBlog/index.html](http://www.lovebxm.com/2017/05/30/buildBlog/index.html "引用链接")里面介绍的比较明白，主要的坑也都说到了。这里说下没提到的坑：</br>
- &emsp;&emsp;```hexo s```是启动本地的server，默认4000端口，但是cmd提示是没有问题，浏览器就是打不开，google后发现可能是foxit占用了4000端口，所以执行```hexo s -p 5000```换个端口就ok了。</br>
- &emsp;&emsp;另外，在markdown的时候，选用```&emsp;```进行汉语空格，```<、br>```进行换行。</br>
- &emsp;&emsp;暂时使用MarkdownPad2进行编译，觉的不是特别理想，有些解析和hexo有出入。明天试试```vscode```或者```haroopad。```</br>
- &emsp;&emsp;在win上的坑比较多，git遇到了用户鉴权怎么都不过，最后到用户管理那里删除github的凭据后重新输入密码才了事。而且```git bash```感觉比较卡，最后还是cmd，不知道win10的WSL会不会好点。


