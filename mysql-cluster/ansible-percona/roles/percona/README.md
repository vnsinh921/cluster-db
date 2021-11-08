Percona XtraDB Cluster
======================

- Cài đặt Percona XtraDB Cluster 5.7 từ Percona apt Repository (Tested on Ubuntu 16.04)
- Khai báo các biến trong defaults/main.yml vào host/group_vars
- Lưu ý biến: `bootstrapped`: Mặc định là `no` khi cluster chưa được bootstrap
  Sau khi đã bootstrap cluster, để thay đổi cấu hình của percona cluster, cần set biến này thành `yes` khi chạy playbook

Example Playbook
================

```
- hosts: percona
  roles:
    - Percona_XtraDB_Cluster
```

- Sau khi cài đặt xong cần thực hiện bootstrap cluster (làm 1 lần khi cài đặt mới) sử dụng role bootstrap_database.


Example Playbook
================

```
- hosts: percona
  roles:
    - bootstrap_database
```
