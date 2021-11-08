# Mồ hình 3 node cài đặt mysql master-mast băng percona-xtradb-cluster
    node-1: 10.0.0.11
    node-2: 10.0.0.12
    node-3: 10.0.0.13
<img src="./images/cau-hinh-Percona-Xtradb-Cluster-8.0-tren-Centos-7-1.png" />
# Thực hiện cài đặt các packge all node
    apt update -y
    apt remove apparmor -y # Remove apparmor
    wget https://repo.percona.com/apt/percona-release_latest.generic_all.deb #Download Percona software repository
    dpkg -i ./percona-release_latest.generic_all.deb #Install Percona software repository
# Run the percona-release all node
    percona-release disable all
    percona-release setup pxc57
    percona-release enable tools release
    apt update -y
# Configures for easy access as root user all node
    touch /root/.my.cnf
    chmod 0400 /root/.my.cnf
    cat <<EOF>> /root/.my.cnf
    [client]
    user     = root
    password = 4E6fPwVY1@5TWERj2IRf
    EOF
# Install percona-server all node
    apt install percona-xtradb-cluster-full-57 python3-mysqldb automysqlbackup -y
# Nhập root password mysql all node
<img src="./images/percona-config.png" />

# Config wsrep percona mysql trên các node
    cp /etc/mysql/percona-xtradb-cluster.conf.d/wsrep.cnf /etc/mysql/percona-xtradb-cluster.conf.d/wsrep.cnf.org
    truncate -s 0 /etc/mysql/percona-xtradb-cluster.conf.d/wsrep.cnf
    cat <<EOF>>/etc/mysql/percona-xtradb-cluster.conf.d/wsrep.cnf
    [mysqld]
    datadir = /var/lib/mysql
    user = mysql
    default-time-zone = '+7:00'
    max_connections = 500
    #bind-address = 10.0.0.11 # Thay đổi IP đúng với từng node
    slow-query-log = /var/log/mysql/mysql-slow.log
    long_query_time = 2

    # UTF8
    default-storage-engine = innodb
    innodb_file_per_table = on
    collation-server = utf8_general_ci
    init-connect = 'SET NAMES utf8'
    character-set-server = utf8
    default-storage-engine = innodb
    innodb_file_per_table = on
    init-connect = 'SET NAMES utf8mb4'
    character-set-client-handshake = FALSE
    character-set-server = utf8mb4
    collation-server = utf8mb4_unicode_ci
    innodb_buffer_pool_size = 1G
    innodb_buffer_pool_instances = 4
    innodb_log_file_size = 128M
    innodb_flush_log_at_trx_commit = 2
    innodb_flush_method = O_DIRECT
    innodb_lru_scan_depth = 256
    innodb_io_capacity = 500
    innodb_io_capacity_max = 2000
    innodb_thread_concurrency = 0
    innodb_read_io_threads = 6
    innodb_write_io_threads = 6
    show_compatibility_56 = ON;
    max_allowed_packet = 1024M
    innodb_autoinc_lock_mode = 2
    pxc_strict_mode = DISABLED

    # Path to Galera library
    wsrep_provider = /usr/lib/galera3/libgalera_smm.so
    # Cluster connection URL contains IPs of node#1, node#2 and node#3
    wsrep_cluster_address = gcomm://10.0.0.11,10.0.0.12,10.0.0.13


    # In order for Galera to work correctly binlog format should be ROW
    binlog_format = ROW
    # MyISAM storage engine has only experimental support
    default_storage_engine = InnoDB
    # This changes how InnoDB autoincrement locks are managed and is a requirement for Galera
    innodb_autoinc_lock_mode = 2
    # Node #2 address
    wsrep_node_address = 10.0.0.11 # thay đổi IP tương ứng từng node
    # Cluster name
    wsrep_cluster_name = Mysql_cluster_test
    # SST method
    wsrep_sst_method = xtrabackup-v2
    #Authentication for SST method
    wsrep_sst_auth = "sstuser:QPrw3Qz0FpxrsF5XU7wI"
    EOF
# Stop mysql on all node
    /etc/init.d/mysql stop
# Bootstrap on master node
    /etc/init.d/mysql bootstrap-pxc
<img src="./images/bootstrap-pxc.png" />

# Create sst user (thực hiện trên master node)
    mysql
    CREATE USER 'sstuser'@'%' IDENTIFIED BY 'QPrw3Qz0FpxrsF5XU7wI';
    GRANT RELOAD,LOCK TABLES,PROCESS,REPLICATION CLIENT ON *.* TO 'sstuser'@'%';
    FLUSH PRIVILEGES;
<img src="./images/create_user_sstuser.png" />

# Restart mysql on other nodes
    /etc/init.d/mysql restart

# Test đồng bộ dữ liệu trên các node
# Tạo database test_node_1 trên node-1:
    mysql
    create database test_node_1;
    use test_node_1
    create table user (c int);
    insert into user (c) values (1);
    select * from user;

# Tạo database test_node_2 trên node-2:
    mysql
    create database test_node_2;
    use test_node_2
    create table user2 (c int);
    insert into user2 (c) values (1);
    select * from user2;
# Tạo database test_node_3 trên node-3:
    mysql
    create database test_node_3;
    use test_node_3
    create table user3 (c int);
    insert into user3 (c) values (1);
    select * from user3;
# Kiểm tra các database trên tất cả các node, nếu đủ 3 database test_node_1,test_node_2,test_node_3 và bản ghi trong table là cluster đã được đồng bộ từ các tất cả các node
    mysql
    show databases;
    use test_node_1
    select * from user;
    use test_node_2
    select * from user2;
    use test_node_3
    select * from user3;   
<img src="./images/node-1.png" />
<img src="./images/node-2.png" />
<img src="./images/node-3.png" />

# Kiểm tra sự động bộ của node sau mất kết nối và join lại cluster
# Stop mysql trên node-1
    /etc/init.d/mysql stop 
# Thực hiện tạo dữ liệu trên database test_node_3 trên node-3
    mysql
    use test_node_3
    create table tbl3 (col1 int);
    insert into tbl3 (col1) values (1000);
    select * from tbl3;

# Shutdown server node-1: 
    init 0
# Start lại server node-1 và check đồng bộ dữ liệu
    mysql
    use test_node_3
    select * from tbl3;
<img src="./images/rsync_after_shutdown.png" />

# Check logs trên node-2 hoặc node-3 để xem chi tiết
    less -f /var/log/mysqld.log







