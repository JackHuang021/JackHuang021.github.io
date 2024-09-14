---
title: Linux下的一些实用操作
tags:
---

### 1. 使用md5sum比较两目录的文件差异
```bash
# 获取第1个文件夹中所有文件的md5值
find ./dir1 -type f -exec md5sum {} \; >> dir1_md5.txt
# 获取第2个文件夹中所有文件的md5值
find ./dir2 -type f -exec md5sum {} \; >> dir2_md5.txt

# 对两个txt文件进行对比
vimdiff dir1_md5.txt dir2_md5.txt
```