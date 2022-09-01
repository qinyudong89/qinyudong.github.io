---
title: "idea git回退到某个历史版本"
date: 2022-08-31T11:20:30-08:00
categories:
  - blog
tags:
  - git
  - idea
---


#### step1.找到要回退的版本号

(右击项目--> Git --> Show History -->选中要回退的版本-->Copy Revision Number)

#### step2.打开idea的Terminal 输入命令

git reset --hard 要回退到的版本号

```
示例：
git reset --hard 139dcfaa558e3276b30b6b2e5cbbb9c00bbdca96
```

#### step3.把修改推到远程服务器

git push -f -u origin 分支

```
示例：
git push -f -u origin dev
```

![img](https://img-blog.csdn.net/20180920181035573?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTA4MDA5NzA=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70&ynotemdtimestamp=1662002811112)