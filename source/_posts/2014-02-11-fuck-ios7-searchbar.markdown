---
title: fuck iOS7 searchBar
date: 2014-02-11 17:53:30
---

今天早上心血来潮想给InstaFollow加一个search当前app里用户功能，不然想找一个特定的美女要TM滚好久。

根据官方文档，一切都进行的很顺利。一般都用searchbar与UISearchDisplayController这对组合。

在你的viewcontroller中：

1. 有一个searchbar属性对象。
2. 根据这个searchbar再有一个UISearchDisplayController对象。
3. 有一个self.searchResultsArray来做UISearchDisplayController tableview的datasource。

这时候你的viewcontroller可以对这对组合负责4件事情。

1. 成为searchDisplayController里面tableview的datasouce（显示资源）。
2. 成为searchDisplayController里面tableview的delegate（响应点击）。
3. searchDisplayController这个controller的delegate, 一般就是searchDisplayController: shouldReloadTableForSearchString: 这个方法如果没有scope（下面的segcontrol）的话。在这个方法里需要更新self.searchResultsArray，然后return YES来自动reload 结果tableview。

所有这一切在iOS6下工作的很正常，在iOS7下，有各种毛病。


前两天连续更新了两个版本的instaLikes。主要原因是appkey，secrect被instagram封杀了，爸爸还封杀了API like的接口。

已经奄奄一息了。

有点厌倦不停的hack instagram。

昨天又把用户的instagram账号密码给破解了。实在不想走到这一步。
