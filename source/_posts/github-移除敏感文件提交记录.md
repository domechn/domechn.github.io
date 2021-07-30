---
title: github 移除敏感文件提交记录
subtitle: github-sensitive
date: 2019-02-28 14:52:51
tags:
- git
- github
---

在往github提交代码时，有可能会将一些私密信息提交到仓库，即使后续删除该文件，该文件的内容依然可以在github的提交记录中被找到。

因此如果需要数据从history中删除，可以使用`git filter-branch`命令或BFG Repo-Cleaner开源工具。

本文主要介绍如何使用`git filter-branch`命令清除history中指定文件的内容。

值得注意的是，如果我们清除了history中的记录，后续我们将无法使用其他命令查看该文件的提交记录和变化。

下面开始使用git filter-branch清除文件提交记录

1、进入项目根路径

```bash
cd YOUR-REPOSITORY
```

2、执行`git filter-branch`

`注意执行该命令后相应的文件也会被删除，如果不希望文件被删除可以先做好备份，后续再恢复`

```bash
git filter-branch --force --index-filter \
'git rm --cached --ignore-unmatch PATH-TO-YOUR-FILE-WITH-SENSITIVE-DATA' \
--prune-empty --tag-name-filter cat -- --all
```

将`PATH-TO-YOUR-FILE-WITH-SENSITIVE-DATA`替换成文件相对于项目的路径

3、将文件添加到`.gitignore`防止文件再次被提交

```bash
echo "YOUR-FILE-WITH-SENSITIVE-DATA" >> .gitignore
git add .gitignore
git commit -m "Add YOUR-FILE-WITH-SENSITIVE-DATA to .gitignore"
```

4、如果你确保不会`push`不会发生冲突，你可以使用

```bash
git push origin --force --all
```

来更新你的所有分支

5、如果需要从标记的版本删除敏感信息你可以使用

```bash
git push origin --force --tags
```

至此即可将敏感文件移除

[官方指导](https://help.github.com/articles/removing-sensitive-data-from-a-repository/)
