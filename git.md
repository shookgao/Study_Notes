**设置用户**
```
git config --global user.name "Your Name"
git config --global user.email "Your Email"
```
**仓库初始化**
```
git init
```

**查看仓库状态**
```
git status
```
**查看不同**
```
git diff <filename>
```
**查看历史提交**
```
git log //可以加参数
```
**查看历史操作**
```
git reflog
```
**版本提交到暂存区**
```
git add <filename>
```
**版本提交**
```
git commit -m "note"
```
**版本回退**
```
git reset --hard <commit_id>
```
**删除文件**
```
git rm <filename>
```
**查看分支**
```
git branch
```
**创建分支**
```
git branch <name>
```
**选择分支**
```
git checkout name
```
**创建加切换分支**
```
git checkout -b <name>
```
**合并分支**
```
git merge <name>
```
**删除分支**
```
git checkout -d <name>
```
**删除远程分支**
```
git push origin --delete <branchname>
```
**查看分支合并图**
```
git log --graph
```
**添加远程仓库**
```
git remote add <shortname> <url>
```
**移除远程仓库**
```
git remote rm <shortname>
```
**克隆远程分支**
```
git clone <url>
```
**拉取远程分支新文件**
```
git pull
```
**推送新文件到远程分支**
```
git push origin master
```
