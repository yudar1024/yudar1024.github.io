+++
author = "陈 sir"
categories = ["software-knowledge"]
tags = ["vsc","idea","sublimetext"]
date = "2019-09-17"
description = "software-knowledge"
featured = "main/title.png"
featuredalt = ""
featuredpath = ""
linktitle = ""
title = "日常软件使用技巧"
type = "post"

+++

# 换行符问题(LF/CRLF)
将VScode， idea ， sublime text 设置为默认使用 linux 下的换行符

1. vscode： 设置--》用户设置--》文本编辑器--》文件--》eol， 设置为`\n` 或者搜索 `files:eol` 进行设置
2. idea： File--》Settings--》Editor--》Code Style--》Line separator 设置为 `Unix and OS X（\n）`
3. sublime text： Perference->Setting-User 中加入配置 `"default_line_ending": "unix"` 可选项为 system， windows， unix
