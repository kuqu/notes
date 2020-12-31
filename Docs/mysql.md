
# mysql基本使用

{docsify-updated}

### 安装

```
apt install mariadb-server
```

### 密码

安装后默认没有密码，可使用`mysql`直接登录。

初始化密码：

```
mysqladmin -u root password “newpass”
```

修改密码：

```
mysqladmin -u root password oldpass “newpass”
```

### 用户

添加用户

```
create user zhangsan identified by “123456”;
```

删除用户

```
DROP USER 'usernamexxx'@'hostxxx';
```

### 权限

添加权限

```
grant all on dbname.* to ‘jack’@‘%’;
```

查看权限

```
show grants for 'gogs’;
```

移除权限

```
REVOKE all ON databasenamexxx.tablenamexxx FROM 'usernamexxx'@'hostxxx';
```

更新权限：

```
flush privileges;
```

