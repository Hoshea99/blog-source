---
title: linux常用命令
date: 2025-10-13 15:11:28
categories: linux
---

## 文件和目录操作

```bash
pwd       # 显示当前目录
cd        # 切换目录
ls        # 列出目录内容
cp        # 复制文件/目录
    cp file.txt backup/
    cp -r dir1/ dir2/
mv        # 移动文件/目录，也用于重命名
	mv 源文件 目标文件
touch     # 创建空文件或更新文件时间戳
mkdir     # 创建目录
rmdir     # 移除空目录
rm        # 删除文件/目录
find      # 查找文件
	find 路径 选项 表达式
	find /home -name "*.txt"
which     # 查找命令位置
whereis   # 查找程序相关文件
```

# 文件查看和处理

```bash
cat       # 显示文件内容
    cat tmp.txt
    cat -n tmp.txt 显示行号
less/more # 分页查看文件
    less large_file.log #支持翻页。less比more好用，不介绍more了
head      # 查看文件前几行
tail      # 查看文件后几行
wc        # 统计文件行数、字数、字节数
diff      # 比较文件差异
grep      # 搜索文本
    grep "error" server.log #直接查
    grep -r "404" /var/log  #递归查找目录
```

# 系统监控

```bash
ps        # 查看进程
    ps -ef | grep nginx
top/htop  # 系统进程监控
kill      # 终止进程
    kill -9 5678
free -h   # 查看剩余内存
df -h     # 查看磁盘使用情况
du -sh    # 查看目录大小
netstat   # 网络连接状态
ss        # 更现代的网络状态查看
```

# 网络

```bash
ping      # 测试网络连通性
telnet    # 测试端口连通性
curl      # 发送HTTP请求
wget      # 下载文件
nc/netcat # 网络调试工具
ifconfig/ip # 网络接口配置
```

# 权限设置

```bash
chmod     # 修改文件权限
    chmod 777 config.conf
    chmod (u/g/o/a)+(r/w/x) script.sh
```

# 压缩

```bash
tar       # 打包压缩
    tar -zcvf log.tar.gz /var/log
    tar -xvf archive.tar -C /var/log
```

