Khai báo biến cho role:

Danh sách các server chỉ sử dụng cho việc đọc dữ liệu
```
sql_read_only_servers: 
  - name: slave1
    address: 127.0.0.1
    port: 3306
```
Danh sách các server sử dụng cho việc đọc, ghi dư liệu

```
sql_read_write_servers:
  - name: master1
    address: 127.0.0.1
    port: 3306
```
Thông tin mysql user kết nối tới cụm  mysql
```
maxscale_db_user_name: myuser
maxscale_db_user_password: mypwd
```

Thêm quyền cho user maxscale ở DB như sau:
```
CREATE USER 'maxscaleuser'@'%' IDENTIFIED BY 'reducetheamount';
GRANT SELECT ON mysql.user TO 'maxscaleuser'@'%';
GRANT SELECT ON mysql.db TO 'maxscaleuser'@'%';
GRANT SELECT ON mysql.tables_priv TO 'maxscaleuser'@'%';
GRANT SELECT ON mysql.roles_mapping TO 'maxscaleuser'@'%';
GRANT SELECT ON mysql.proxies_priv TO 'maxscale'@'%';
GRANT SHOW DATABASES ON *.* TO 'maxscaleuser'@'%';
GRANT REPLICATION CLIENT on *.* to 'maxscaleuser'@'%';
```