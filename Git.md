#Git设置Https或Http用户名密码保存
 

当我们使用git clone命令去克隆一个远程仓库的时候（使用Http或者Https的方式），会每次pull和push操作的时候都会输入一次用户名和密码操作。接下来我们来看一下几种记住用户名和密码的方式。

1.长期存储密码：

 `git config --global credential.helper store`

2.设置记住密码（默认是15分钟）：

    git config --global credential.helper cache

如果想自己设置时间，可以这样去实现这个：

    git config credential.helper 'cache --timeout=3600'

这样就设置一个小时之后失效

增加远程地址的时候带上密码也是可以的。(推荐)

    http://yourname:password@github.com/projectname.git

## 刪除遠端分支 ##
如果不再需要某個遠端分支了，比如搞定了某個特性並把它合併進了遠端的 master 分支（或任何其他存放穩定代碼的分支），可以用這個非常無厘頭的語法來刪除它：git push [遠程名] :[分支名]。如果想在服务器上刪除 serverfix 分支，执行下面的命令：

	$ git push origin :serverfix
	To git@github.com:schacon/simplegit.git
	 - [deleted] serverfix