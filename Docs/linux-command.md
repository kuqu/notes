
# Linux 常用命令

{docsify-updated}

### 添加系统用户用于启动程序

创建系统组
```
addgroup --system gogs
cat /etc/group #查看是否创建成功
```

创建系统用户
```
adduser --system --ingroup gogs --no-create-home --disable-passowrd gogs
usermod -c "用户运行Gogs服务" -d /usr/local/Gogs -g gogs gogs
```

修改程序所属用户
```
chown -R gogs:gogs /usr/local/gogs # 递归（-R）设置目录所有者和所有组
chmod u+rwx,g+rxs,o= /usr/local/Gogs  # 修改目录权限
```





