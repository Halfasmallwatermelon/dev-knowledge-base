# PM2 进程管理

## 什么是 PM2？

PM2 是 Node.js 的进程管理器，支持守护进程、负载均衡、日志管理。

## 常用命令

```bash
# 启动应用
pm2 start app.js

# 启动并命名
pm2 start app.js --name myapp

# 查看状态
pm2 status

# 查看日志
pm2 logs

# 监控
pm2 monit

# 重启
pm2 restart myapp

# 停止
pm2 stop myapp

# 删除
pm2 delete myapp
```

## 配置文件

```javascript
// ecosystem.config.js
module.exports = {
  apps: [{
    name: 'myapp',
    script: './app.js',
    instances: 'max',
    exec_mode: 'cluster',
    env: {
      NODE_ENV: 'production',
      PORT: 3000
    },
    log_date_format: 'YYYY-MM-DD HH:mm:ss',
    error_file: './logs/error.log',
    out_file: './logs/out.log'
  }]
}

// 启动
pm2 start ecosystem.config.js
```

## 集群模式

```bash
# 启动4个实例
pm2 start app.js -i 4

# 或使用 max（CPU核心数）
pm2 start app.js -i max
```

## 日志管理

```bash
# 查看日志
pm2 logs

# 清空日志
pm2 flush

# 日志轮转
pm2 install pm2-logrotate
```

## 启动脚本

```bash
# 生成启动脚本
pm2 startup

# 保存当前进程列表
pm2 save

# 恢复进程
pm2 resurrect
```