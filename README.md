**前言：**

​    生产上有时需要根据指定内容查找相关文件，比如FastJson反序列化漏洞，通过'FastJson'关键字查找有无对应文件，如果有则进行整改。

**环境说明：**

|   主机名    |  操作系统版本   |      ip      | ansible version |       备注        |
| :---------: | :-------------: | :----------: | :-------------: | :---------------: |
|   ansible   | Centos 7.6.1810 | 172.27.34.51 |      2.9.9      | ansible管理服务器 |
| ansible-awx | Centos 7.6.1810 | 172.27.34.50 |        /        |    被管服务器     |

## 一、文件列表

```bash
[root@ansible-awx ~]# cd /opt
[root@ansible-awx opt]# ls -alrt
总用量 12
drwx--x--x   4 root root  28 5月  21 13:50 containerd
dr-xr-xr-x. 19 root root 250 7月  14 11:07 ..
drwxr-xr-x   2 root root  38 8月   6 15:34 find1
drwxr-xr-x   2 root root  38 8月   6 15:35 find2
drwxr-xr-x   2 root root  38 8月   6 15:35 find3
-rw-r--r--   1 root root  12 8月   6 15:37 test1.txt
-rw-r--r--   1 root root  12 8月   6 15:37 test2.txt
-rw-r--r--   1 root root  12 8月   6 15:37 .test3.txt
drwxr-xr-x.  6 root root 115 8月   6 15:40 .
[root@ansible-awx opt]# find .  -path ./containerd  -prune -o  -type f |xargs grep  error
grep: ./containerd: 是一个目录
./find1/find1.txt:aaaerrorbbb
./find1/.a1.txt:aaaerrorbbb
匹配到二进制文件 ./find2/find2.txt
./find2/.a2.txt:aaaerrorbbb
./find3/find3.txt:aaaerrorbbb
./find3/.a3.txt:aaaerrorbbb
./test1.txt:aaaerrorbbb
./.test3.txt:aaaerrorbbb
./test2.txt:aaaerrorbbb
[root@ansible-awx opt]# tail -n2 find2/find2.txt
aaaerrorbbb
[root@ansible-awx opt]# du -sm find2/*
201     find2/find2.txt
[root@ansible-awx opt]# tree -a
.
├── containerd
│   ├── bin
│   └── lib
├── find1
│   ├── .a1.txt
│   └── find1.txt
├── find2
│   ├── .a2.txt
│   └── find2.txt
├── find3
│   ├── .a3.txt
│   └── find3.txt
├── test1.txt
├── test2.txt
└── .test3.txt

6 directories, 9 files
```

![image-20200806163829302](https://i.loli.net/2020/08/06/QK2IeH1yMrR9N6X.png)

在被管服务器test50的/opt目录下构造测试数据，find1、find2、find3为目录，test1.txt、test2.txt为文件，.test3.txt为隐藏文件，3个find目录都有两个文件，一个txt文件和一个隐藏文件，这些文件都包含字符串'aaaerrorbbb'，其中find2目录的find2.txt大小为201M。

## 二、role总览

### 1.初始化role

```bash
[root@ansible roles]# ansible-galaxy init find
- Role find was created successfully
```

### 2.执行文件

```bash
[root@ansible ~]# cd /etc/ansible
[root@ansible ansible]# more find.yaml 
---
- hosts: "{{ hostlist }}"
  gather_facts: no
  roles:
  - role: find 
```

role名为find，hosts列表需执行的时候指定。

### 3.task文件

```bash
[root@ansible ansible]# more roles/find/tasks/main.yml 
---
# tasks file for find
# author: loong576

- name: choose the directory 
  find:
    paths: "{{ directory_path }}" 
    recurse: no
    file_type: directory
    excludes: "{{ exclude_directory }}" 
  register: find_directory

- name: find in directory 
  find:
    paths: "{{item.path}}" 
    recurse: yes 
    contains: "{{ file_contains }}" 
    hidden: yes
    size: "{{ file_size }}" 
  with_list: "{{find_directory.files}}"
  register: find_contains_in_directory


- name: echo find file in directories
  debug:
    msg:
      "{% for i in item.files %}
          {{ i.path }} 
       {% endfor %}"
  with_list: "{{find_contains_in_directory.results}}"
  when: item.matched != 0

- name: find in files
  find:
    path: "{{ file_path }}" 
    file_type: file
    excludes: "{{ exclude_file }}" 
    hidden: yes
    contains: "{{ file_contains }}" 
  register: find_only_file

- name: echo find file in files
  debug:
    msg: "{{item.path}}"
  with_list: "{{find_only_file.files}}"
```

#### 执行逻辑

**指定路径下目录查找**

首先选择需要查找的指定路径{{ directory_path }}，这里为/opt，选择的时候排除掉不需要的目录{{ exclude_directory }}；然后通过循环方式在选择的目录里查找指定内容{{ file_contains }}并输出查到的文件列表。

这里的目录指/opt下的find1和find2，find3被排除在外。

**指定路径下文件查找**

查找指定路径{{ directory_path }}下所有文件是否包含指定内容{{ file_contains }}并输出文件列表，{{ exclude_file }}文件被排除在外。

这里的文件指test1.txt、.test3.txt，test2.txt被排除在外。

所有的隐藏文件默认被查找'hidden: yes'且找到的文件大小不能超过{{ file_size }}即100M

### 4.default文件

```bash
[root@ansible ansible]# more roles/find/defaults/main.yml 
---
# defaults file for find
directory_path: /opt
exclude_directory: find3
file_path: /opt
file_contains: .*error.*
exclude_file: test2.txt
file_size: -100m
```

指定查找的内容为带有'error'的文件，指定的路径为/opt，排查的目录为find3，排除的文件为test2.txt，所有查找的文件大小小于100m。

## 三、运行role

### 1.预期

/opt下的目录find1的文件find1.txt和隐藏文件.a1.txt被输出；目录find2的隐藏文件.a2.txt被输出；/opt下的文件test1.txt和隐藏文件.test3.txt被输出；被排除的目录find3和被排除的文件test2.txt将不会被输出；不满足大小要求的find2.txt也不会被输出。

### 2.执行role

```bash
[root@ansible ~]# cd /etc/ansible/
[root@ansible ansible]# ansible-playbook find.yaml -e hostlist=test50
```

指定主机列表为test50

![image-20200806164758638](https://i.loli.net/2020/08/06/jBMF463wtLfe8pV.png)

结果符合预期

&nbsp;

&nbsp;


**更多请点击：**[ansible系列文章](https://blog.51cto.com/3241766/category17.html)

