---
layout: post
title: How to work with GitHub and Multiple Accounts
tags: [git]
categories: [Github]
---


前段时间剁手买了Macbook来开发,用起来还是蛮爽的，顺便在Mac配置公司的gitlab帐号。今天心xue来潮想用github搭建自己的博客，在mac上再配置了一个新的GitHub帐号，下面记一下笔记：

Step1:新建一个SSH Key

关联账号

{% highlight ruby %}
buglaideMacBook-Pro:~ buglai$ ssh-keygen -t rsa -C "buglai666@gmail.com"
{% endhighlight %}

保存新账号的key，注意命名不能和原来的重复，否则会覆盖，这样就保存了密钥

{% highlight ruby %}
Enter file in which to save the key (/Users/buglai/.ssh/id_rsa): /Users/buglai/.ssh/id_rsa_second
{% endhighlight %}

进入ssh目录

{% highlight ruby %}
buglaideMacBook-Pro:~ buglai$ cd ~/.ssh
{% endhighlight %}


查看目录文件，发现多了新的key

{% highlight ruby %}
buglaideMacBook-Pro:.ssh buglai$ ls -la
total 40
drwx------ 7 buglai staff 238 12 27 15:42 .
hello there
drwxr-xr-x+ 30 buglai staff 1020 12 27 15:30 ..
-rw------- 1 buglai staff 1679 9 7 18:23 id_rsa
-rw-r--r-- 1 buglai staff 399 9 7 18:23 id_rsa.pub
-rw------- 1 buglai staff 1675 12 27 15:42 id_rsa_second
-rw-r--r-- 1 buglai staff 401 12 27 15:42 id_rsa_second.pub
-rw-r--r-- 1 buglai staff 409 9 7 18:28 known_hosts
{% endhighlight %}


到github页面登录，点击导航栏Account Settings ，点击SSH Public Keys,点击Add another public key
打开d_rsa_second.pub，获得key复制到github上面的key方框，记得ssh-rsa也要复制进去，title随便命名

{% highlight ruby %}
buglaideMacBook-Pro:.ssh buglai$ vim id_rsa_second.pub
{% endhighlight %}

点击Add key就行了,Mac上面的key需要进行认证一下

{% highlight ruby %}
buglaideMacBook-Pro:.ssh buglai$ ssh-add ~/.ssh/id_rsa_second
{% endhighlight %}


接下来要新建一个文件来进行多个账号的配置，使得我们在上传代码的时候能够辨别上传到哪个账号


{% highlight ruby %}
buglaideMacBook-Pro:.ssh buglai$ touch config
buglaideMacBook-Pro:.ssh buglai$ vim config

#Default Gitlub
Host github.com
HostName github.com
User git
IdentityFile ~/.ssh/id_rsa
Host github-BugLai
HostName github.com
User git
IdentityFile ~/.ssh/id_rsa_second
{% endhighlight %}


接下来测试一下是否可用：在github上面点击New repository，name为Test ，点击Create repository显示结果如下：

{% highlight ruby %}
touch README.md
git init
git add README.md
git commit -m "first commit"
git remote add origin https://github.com/BugLai/Testd.git
git push -u origin master

buglaideMacBook-Pro:~ buglai$ cd Desktop/
buglaideMacBook-Pro:Desktop buglai$ mkdir test
buglaideMacBook-Pro:Desktop buglai$ cd test/
buglaideMacBook-Pro:test buglai$ touch readme.md
buglaideMacBook-Pro:test buglai$ vim readme.md
buglaideMacBook-Pro:test buglai$ git init
Initialized empty Git repository in /Users/buglai/Desktop/test/.git/
buglaideMacBook-Pro:test buglai$ git add .
buglaideMacBook-Pro:test buglai$ git commit -m "test github"
[master (root-commit) 90ae46f] test github
1 file changed, 1 insertion(+)
create mode 100644 readme.md
{% endhighlight %}


这里注意因为我们的config文件里面用的Host是github-BugLai所以不能够直接复制
git remote add origin https://github.com/BugLai/Testd.git而要用下面的：
buglaideMacBook-Pro:test buglai$ git remote add origin git@github-BugLai:BugLai/Test.git
回去gitbug上面，可以看到已经成功上传了,当然假如你想要测试原来的账号是否能够正常使用，可以如下

{% highlight ruby %}
buglaideMacBook-Pro:test buglai$ rm -rf .git
buglaideMacBook-Pro:test buglai$ git init
Initialized empty Git repository in /Users/buglai/Desktop/test/.git/
buglaideMacBook-Pro:test buglai$ git add .
buglaideMacBook-Pro:test buglai$ git commit -m 'for gitlib'
buglaideMacBook-Pro:test buglai$ git remote add origin git@git.umlife.net:selfgroup/test.git
buglaideMacBook-Pro:test buglai$ git push -u origin master
{% endhighlight %}

以上也显示成功了，以上成功演示了同一电脑的多github账号管理
http://code.tutsplus.com/tutorials/quick-tip-how-to-work-with-github-and-multiple-accounts--net-22574
