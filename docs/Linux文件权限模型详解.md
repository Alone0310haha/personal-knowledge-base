---
title: Linux文件权限模型详解
date: 2026-03-20
tags: [Linux, 权限, ACL, 文件系统]
category: 操作系统
---

# Linux文件权限模型详解

## 1. 基本权限模型 (UGO)

Linux 最基础的权限模型基于 **所有者(User)-所属组(Group)-其他用户(Other)** 的三维体系。

### 1.1 三类权限

| 权限 | 符号 | 对文件的意义 | 对目录的意义 |
|------|------|--------------|--------------|
| 读 (Read) | `r` | 可以读取文件内容 | 可以列出目录内容 |
| 写 (Write) | `w` | 可以修改文件内容 | 可以在目录中创建/删除文件 |
| 执行 (Execute) | `x` | 可以执行文件 | 可以进入目录 (cd) |

### 1.2 权限表示方法

```bash
# 符号形式
rwxr-xr-x  (755)

# 数字形式
755 = rwxr-xr-x
644 = rw-r--r--
600 = rw-------
777 = rwxrwxrwx
```

- **第一位**：文件类型 (`-` 文件, `d` 目录, `l` 符号链接, `c` 字符设备, `b` 块设备)
- **第2-4位**：所有者权限
- **第5-7位**：所属组权限
- **第8-10位**：其他用户权限

### 1.3 常用命令

```bash
# 修改权限
chmod 755 file.sh           # 数字方式
chmod +x file.sh            # 添加执行权限
chmod u+w file.sh           # 给所有者添加写权限
chmod g-r file.sh           # 删除组读权限

# 修改所有者和所属组
chown user:group file       # 同时修改所有者和组
chown user file             # 只改所有者
chgrp group file            # 只改组

# 查看权限
ls -l file
stat file                   # 详细状态信息
```

## 2. 特殊权限

### 2.1 SetUID (suid)

- 作用：执行时使用文件所有者的权限
- 数值：`4000`
- 标志位：`s` 或 `S`（如果没有 x 权限）
- 典型例子：`/usr/bin/passwd`

```bash
ls -l /usr/bin/passwd
# -rwsr-xr-x 1 root root  ... /usr/bin/passwd
```

### 2.2 SetGID (sgid)

- 作用：执行时使用文件所属组的权限；目录中创建的文件继承目录的组
- 数值：`2000`
- 标志位：`s` 或 `S`
- 典型场景：共享目录

```bash
# 创建组共享目录
mkdir /shared
chmod 2775 /shared
chgrp developers /shared
```

### 2.3 Sticky Bit (粘滞位)

- 作用：目录中只有文件所有者能删除自己的文件
- 数值：`1000`
- 标志位：`t` 或 `T`
- 典型例子：`/tmp`

```bash
ls -ld /tmp
# drwxrwxrwt 10 root root  ... /tmp
```

## 3. ACL (访问控制列表)

传统的 UGO 模型粒度较粗，ACL 提供了更灵活的权限控制。

### 3.1 ACL 相关命令

```bash
# 查看 ACL
getfacl file

# 设置 ACL
setfacl -m u:alice:rw file      # 给用户 alice 读写权限
setfacl -m g:developers:r-x file # 给组读执行权限
setfacl -m o::r file            # 设置其他用户权限 (等同于 chmod)

# 删除 ACL
setfacl -x u:alice file         # 删除指定用户的 ACL
setfacl -b file                 # 删除所有 ACL

# 设置默认 ACL (目录)
setfacl -R -m d:u:alice:rw /shared  # 新文件自动继承
```

### 3.2 ACL 权限检查顺序

1. 如果进程是文件所有者 → 应用所有者权限
2. 如果进程属于文件所属组 → 应用组权限
3. 否则 → 应用其他用户权限
4. 如果以上都不匹配 → 应用 ACL

### 3.3 备份和恢复 ACL

```bash
# 备份
getfacl -R /path > acl_backup.txt

# 恢复
setfacl --restore=acl_backup.txt
```

## 4. 隐藏权限 (chattr)

```bash
# 查看隐藏权限
lsattr file

# 常用属性
chattr +i file      # 不可修改 (即使是 root)
chattr +a file      # 只能追加，不能删除或修改
chattr +S file      # 同步写入
```

## 5. 最佳实践

1. **最小权限原则**：只给必要的最小权限
2. **谨慎使用 777**：除非特殊情况，避免给 777
3. **注意 suid/sgid 风险**：定期检查系统中的 suid 文件
4. **使用 ACL 细化控制**：需要精细控制时使用 ACL
5. **定期审计权限**：`find / -perm -4000` 查找可疑的 suid 文件

---

*📅 创建于：2026-03-20*