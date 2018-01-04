---
title: Linux中C++实现通用读取目录文件代码
tags:
  - C++
categories:
  - C++
date: 2015-03-22 23:47:17
---

Linux中C++实现通用读取目录文件代码
```
#include <dirent.h>
#include <stdio.h>
void createDocList(std::vector<std::string> &doc_list){
    int return_code;
    DIR *dir;
    struct dirent entry;
    struct dirent *res;
    string real_dir = "/helloyou/home/";//搜索的目录
    if ((dir = opendir(real_dir.c_str())) != NULL) {//打开目录
        for (return_code = readdir_r(dir, &entry, &res);res != NULL && return_code == 0;return_code = readdir_r(dir, &entry, &res)) {
            if (entry.d_type != DT_DIR) {//存放到列表中
                doc_list.push_back(string(entry.d_name));
            }
        }
        closedir(dir);//关闭目录
    }
}
```