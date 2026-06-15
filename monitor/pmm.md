### Install Percona Monitorring and Management Server
```bash
# https://docs.percona.com/percona-monitoring-and-management/3/install-pmm/install-pmm-server/deployment-options/docker/index.html
curl -fsSL https://www.percona.com/get/pmm | /bin/bash
```

---

### Add MySQL Monitoring User
```mysql
CREATE USER 'pmm'@'%' IDENTIFIED BY 'UbET57qfzIaMkpwo';
GRANT SELECT, PROCESS, REPLICATION CLIENT ON *.* TO 'pmm'@'%';
ALTER USER 'pmm'@'%' WITH MAX_USER_CONNECTIONS 10;
GRANT SELECT ON performance_schema.* TO 'pmm'@'%';
```
