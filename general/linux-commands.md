# Linux 常用命令

## 文件操作

```bash
# 列出文件
ls -la          # 详细列表
ls -lh          # 人类可读大小

# 创建/删除
mkdir dir       # 创建目录
rm file         # 删除文件
rm -rf dir      # 递归删除目录

# 复制/移动
cp src dest     # 复制
mv src dest     # 移动/重命名

# 查找
find / -name "*.log"      # 按名称查找
grep -r "pattern" /path   # 搜索内容
```

## 系统信息

```bash
# 系统信息
uname -a                    # 系统信息
cat /etc/os-release         # 操作系统信息
free -h                     # 内存使用
df -h                       # 磁盘使用
top                         # 进程监控
htop                        # 增强版top

# 网络
netstat -tulpn              # 网络连接
ss -tulpn                   # socket统计
curl http://example.com     # HTTP请求
ping google.com             # 测试连接
```

## 进程管理

```bash
# 查看进程
ps aux                      # 所有进程
ps aux | grep nginx         # 查找特定进程

# 终止进程
kill PID                    # 终止进程
kill -9 PID                 # 强制终止
pkill nginx                 # 按名称终止

# 后台运行
nohup command &             # 后台运行
jobs                        # 查看后台任务
fg %1                       # 调回前台
```

## 用户管理

```bash
# 用户操作
useradd username            # 创建用户
passwd username             # 设置密码
usermod -aG sudo username   # 添加到sudo组

# 权限
chmod 755 file              # 修改权限
chown user:group file       # 修改所有者
```

## 服务管理

```bash
# systemctl
systemctl start nginx       # 启动服务
systemctl stop nginx        # 停止服务
systemctl restart nginx     # 重启服务
systemctl status nginx      # 查看状态
systemctl enable nginx      # 开机自启
```

## 压缩/解压

```bash
# tar
tar -czf archive.tar.gz dir   # 压缩
tar -xzf archive.tar.gz       # 解压

# zip
zip -r archive.zip dir        # 压缩
unzip archive.zip             # 解压
```