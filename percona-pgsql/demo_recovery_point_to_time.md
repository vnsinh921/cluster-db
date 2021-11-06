# Demo recovery point to time
# Tạo database demo
    psql
    postgres=# create database demo;
<img src="./images/Screenshot 2021-11-06 214813.png" />

# Switch sang database demo, tạo table tbl1 và insert 1000 role 
    postgres=# \c demo
    demo=# create table tbl1 (col1 int);
    demo=# insert into tbl1 (col1) select generate_series(1, 10000);
    demo=# select count(*) from tbl1;
    demo=# select now(); 
# Tại thời điểm 2021-11-06 17:04:52 trên db demo có 10000 bản ghi   
<img src="./images/Screenshot 2021-11-06 215409.png" />

# Tiến hành backup lại db bằng  pg_basebackup
    pg_basebackup -U repmgr -D - -Ft -X fetch > postgresql_$(date '+%Y_%m_%d_%H_%M_%S').tar.bz2
    ls -lh |grep bz2
<img src="./images/Screenshot 2021-11-06 220723.png" />

# Tiếp tục insert thêm 1000000 vào table tbl1 db demo
    psql
    postgres=# \c demo
    demo=# insert into tbl1 (col1) select generate_series(10001, 1000000);
    demo=# select count(*) from tbl1;
    demo=# select now();

# Tại thời điểm 2021-11-06 17:25:30 trên db demo có 10000 bản ghi  
<img src="./images/Screenshot 2021-11-06 221149.png" />

# Bây giờ tiến hành recovery lại database tại thời điểm 2021-11-06 17:04:52
    mkdir -p /var/lib/pgsql/recovery
    chmod -R 700 /var/lib/pgsql/recovery
    cd /var/lib/pgsql/recovery
    cp /var/lib/pgsql/postgresql_2021_11_06_17_19_41.tar.bz2 ./
    tar xvf postgresql_2021_11_06_17_19_41.tar.bz2 # Giai nen file backup
    ls
<img src="./images/Screenshot 2021-11-06 222459.png" />

# Tạo file /var/lib/pgsql/recovery/recovery.conf
    touch recovery.conf
    cat <<EOF>> /var/lib/pgsql/recovery/recovery.conf
    restore_command = 'cp /var/lib/pgsql/9.6/archives/%f "%p"'
    recovery_target_time = '2021-11-06 17:04:52'
    EOF
# Tìm dòng #port = 5432 của file  /var/lib/pgsql/recovery/postgresql.conf thành port = 5400
    sed -i 's/#port = 5432/port = 5400/g' /var/lib/pgsql/recovery/postgresql.conf # Mục đích start data recovery ở port 5400 
# Start database
    pg_ctl -D /var/lib/pgsql/recovery -l recovery.log start
    netstat -npl |grep 5400
<img src="./images/Screenshot 2021-11-06 224025.png" />

# Check log recovery
    cat /var/lib/pgsql/recovery/recovery.log
<img src="./images/Screenshot 2021-11-06 224211.png" />

# Vậy là ta đã hoàn thành recovery database về thời điểm 2021-11-06 17:04:52
# Kiểm Tra tính chính xác của dữ liệu như sau, nếu kết quả như hình bên dưới là đã thành công:
    psql -p 5400
    postgres=# \c demo
    demo=# select count(*) from tbl1;
<img src="./images/Screenshot 2021-11-06 224457.png" />

# Số lượng bản ghi đã được khôi phục chính xác về thời điểm 2021-11-06 17:04:52
