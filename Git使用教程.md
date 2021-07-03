初次配置：

git config --global user.name "用户名"

git config --global user.email "邮箱"

git remote add origin 库地址.git 

git remote rm origin 

git push -u origin master

理论基础：Git--将每个版本文件独立保存

​					  三棵树：工作区域（init）				暂存区域					Git仓库

​																		>add>				>commit>

​					  文件状态：已修改								已暂存						已提交

查看状态：git status  对比三个区域

​					 git reset Head 撤销上一次的提交，可后接<文件名>,**3->2**

​					 git checkout -- <文件名> 工作区域file覆盖暂存，且删除前者

​					 git log 历史提交记录

回到过去：log随之改变

​					 git reset HEAD~n 仓库指向上n次的提交

​					 git reset --mixed HEAD~ 仓库指向改变且暂存区域同步

​					 git reset --soft HEAD~ 仓库指向上一次

​					 git reset --hard HEAD~ 仓库指向上一次，且工作、暂存区域均同步

​					 git reset ID号  仓库指向指定版本

​					 git reset ID号 文件名/路径  仓库回滚个别文件

版本对比：

​					 git diff  比较工作区域与暂存区域文件

​					 git diff ID1 ID2 比较两次提交

​					 git diff  ID 比较工作区域与该版本，最新版本为HEAD

​					 git diff --cached ID 比较暂存区与该ID版本， 最新则不加ID

修改最后一次提交：

​					 git commit -amend 新开界面进行暂存区readme更改

​					 git rm --cached 文件名 删除暂存区域的文件，版本记录需reset， -f则删两区

​					 git mv 旧文件名 新文件名    重命名

​					 git add  新文件名

创建和切换分支（另一个完整文件）：

​					 git branch 分支名   创建分支

​					 git checkout 分支名 切换分支， 工作区域也会被覆盖

合并和删除分支：

​					 git merge 分支名  合并分支至当前分支，替代目前分支内容

​					 git branch -d 分支名  删除该分支



