# Configure source server(base GTID)
##### 使用mysql超级用户登陆source mysql，执行以下命令
```
# 安装clone插件
INSTALL PLUGIN clone SONAME 'mysql_clone.so';

# 创建clone用户，用于克隆源实例
CREATE USER 'clone'@'%' IDENTIFIED BY 'kE6Qqg5Ga1ANh0+tCqfF';
GRANT BACKUP_ADMIN ON *.* TO 'clone'@'%';

# 创建replica用户，用于同步源实例
CREATE USER replica@'%' IDENTIFIED BY 'Faw1gt2wtikpuf597Hrw_';
GRANT REPLICATION SLAVE,REPLICATION CLIENT  on *.* to replica@'%';
```

# Configure replication server(base GTID)
##### 使用mysql超级用户登陆replication mysql，执行以下命令
```
# 安装clone插件
INSTALL PLUGIN clone SONAME 'mysql_clone.so';

# 设置有效的克隆源列表
SET GLOBAL clone_valid_donor_list = '192.168.1.1:3306'

# 克隆实例
CLONE INSTANCE FROM 'clone'@'192.168.1.11':3306 IDENTIFIED BY 'kE6Qqg5Ga1ANh0+tCqfF';

# 设置主从同步
## 8.4版本前
CHANGE MASTER TO
  MASTER_HOST = '192.168.1.1',
  MASTER_PORT = 3306,
  MASTER_USER = 'replica',
  MASTER_PASSWORD = 'Faw1gt2wtikpuf597Hrw_',
  MASTER_AUTO_POSITION = 1,
  MASTER_RETRY_COUNT = 50,
  MASTER_SSL = 1;

start slave


## 8.4版本后
CHANGE REPLICATION SOURCE TO
  SOURCE_HOST = '192.168.1.1',
  SOURCE_PORT = 3306,
  SOURCE_USER = 'replica',
  SOURCE_PASSWORD = 'Faw1gt2wtikpuf597Hrw_',
  SOURCE_AUTO_POSITION = 1,
  SOURCE_RETRY_COUNT = 50,
  SOURCE_SSL = 1;

start replica
```
