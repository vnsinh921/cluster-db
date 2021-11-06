Hướng dẫn cài đặt postgresql Auto Failover Repmgr  
Mô hình:  
10.0.0.11 Node-1 master (Centos 7)  
10.0.0.12 Node-2 slave-1 (Centos 7)  
10.0.0.13 Node-3 slave-2 (Centos 7)  
10.0.0.14 Haproxy (Ubuntu-20.04)  
1. Chuẩn bị môi trường
- Trên tất cả các node cài đặt postgresql
# Sửa file /etc/hosts

    cat <<EOF>> /etc/hosts
    10.0.0.11 node-1
    10.0.0.12 node-2
    10.0.0.13 node-3
    EOF
# Cài đặt postgresql trên tất cả các node
    yum install https://download.postgresql.org/pub/repos/yum/reporpms/EL-7-x86_64/pgdg-redhat-repo-latest.noarch.rpm -y
    yum install postgresql96 postgresql96-server postgresql96-contrib postgresql96-libs repmgr96 xinetd -y
    ln -s /usr/pgsql-9.6/bin/repmgr /usr/bin/
# Tao file .bashrc trên tất cả các node
    su postgres && cd 
    touch .bashrc 
    cat <<EOF >> .bashrc 
    # .bashrc
    # User specific aliases and functions
    alias rm='rm -i'
    alias cp='cp -i'
    alias mv='mv -i'
    # Source global definitions
    if [ -f /etc/bashrc ]; then
            . /etc/bashrc
    fi
    source ~/.bash_profile
    EOF
# Tao file .bash_profile trên tất cả các node
    su postgres && cd 
    touch .bash_profile 
    cat <<EOF >> .bash_profile
    [ -f /etc/profile ] && source /etc/profile
    PGDATA=/var/lib/pgsql/9.6/data
    export PGDATA
    export PATH="$PATH:/usr/pgsql-9.6/bin"
    export PS1="[\u@\h \W]\\$ "
    # If you want to customize your settings,
    # Use the file below. This is not overridden
    # by the RPMS.
    [ -f /var/lib/pgsql/.pgsql_profile ] && source /var/lib/pgsql/.pgsql_profile
    alias ssh='ssh -o StrictHostKeyChecking=no'
    alias scp='scp -o StrictHostKeyChecking=no'
    alias rsync='rsync -e "ssh -o StrictHostKeyChecking=no"'
    EOF
# Tao thu muc luu log cho postgres
    mkdir /var/log/postgresql
    chown -R postgres:postgres /var/log/postgresql
# Trên node master
    /usr/pgsql-9.6/bin/postgresql96-setup initdb
# Config postgresql trên node master
sửa các tham số sau trên file: /var/lib/pgsql/9.6/data/postgresql.conf  

        listen_addresses = '*'  
        shared_preload_libraries = 'repmgr'   
        wal_level = replica  
        wal_log_hints = on  
        archive_mode = on
        archive_command = '/var/lib/pgsql/9.6/archives'  
        max_wal_senders = 3  
        max_replication_slots = 2  
        hot_standby = on   
        log_checkpoints = on

# Cấu hình file /etc/repmgr/9.6/repmgr.conf
    cp /etc/repmgr/9.6/repmgr.conf /etc/repmgr/9.6/repmgr.conf_org
    truncate -s 0 /etc/repmgr/9.6/repmgr.conf
    cat <<EOF>> /etc/repmgr/9.6/repmgr.conf
    cluster='pg_cluster'
    node_id=1
    node_name=node-1
    conninfo='host=10.0.0.11  user=repmgr dbname=repmgr connect_timeout=2'
    data_directory='/var/lib/pgsql/9.6/data'
    failover=automatic
    reconnect_attempts=3
    reconnect_interval=5
    promote_command='/usr/pgsql-9.6/bin/repmgr standby promote --log-to-file'
    follow_command='/usr/pgsql-9.6/bin/repmgr standby follow --log-to-file --upstream-node-id=%n'
    log_file='/var/log/repmgr/repmgrd-9.6.log'
    event_notifications='standby_promote'
    pg_bindir='/usr/pgsql-9.6/bin/'
    EOF
# Cấu hình file /var/lib/pgsql/9.6/data/pg_hba.conf
Thêm các dòng sau vào cuối file:  

    cat <<EOF>> /var/lib/pgsql/9.6/data/pg_hba.conf
    local   replication     repmgr                                  trust  
    host    replication     repmgr          127.0.0.1/32            trust  
    host    replication     repmgr          10.0.0.11/32            trust  
    host    replication     repmgr          10.0.0.12/32            trust  
    host    replication     repmgr          10.0.0.13/32            trust  
    host    replication     repmgr          10.0.0.14/32            trust  
    local   repmgr          repmgr                                  trust  
    host    repmgr          repmgr          127.0.0.1/32            trust  
    host    repmgr          repmgr          10.0.0.11/32            trust  
    host    repmgr          repmgr          10.0.0.12/32            trust  
    host    repmgr          repmgr          10.0.0.13/32            trust  
    host    repmgr          repmgr          10.0.0.14/32            trust
    EOF  

# Start service postgresql
    systemctl start postgresql-9.6
    systemctl enable postgresql-9.6
# Primary: Register the primary server
    su postgres
    createuser -s repmgr
    createdb repmgr -O repmgr
    repmgr primary register 
    repmgr cluster show
<img src="./images/Screenshot 2021-11-05 235054.png" />  


# Slave-1
# Cấu hình file /etc/repmgr/9.6/repmgr.conf

    cp /etc/repmgr/9.6/repmgr.conf /etc/repmgr/9.6/repmgr.conf_org
    truncate -s 0 /etc/repmgr/9.6/repmgr.conf
    cat <<EOF>> /etc/repmgr/9.6/repmgr.conf
    cluster='pg_cluster'
    node_id=2
    node_name=node-2
    conninfo='host=10.0.0.12  user=repmgr dbname=repmgr connect_timeout=2'
    data_directory='/var/lib/pgsql/9.6/data'
    failover=automatic
    reconnect_attempts=3
    reconnect_interval=5
    promote_command='/usr/pgsql-9.6/bin/repmgr standby promote --log-to-file'
    follow_command='/usr/pgsql-9.6/bin/repmgr standby follow --log-to-file --upstream-node-id=%n'
    log_file='/var/log/repmgr/repmgrd-9.6.log'
    event_notifications='standby_promote'
    pg_bindir='/usr/pgsql-9.6/bin/'
    EOF
# Clone  and register standby database tu node master

    repmgr -h 10.0.0.11  -U repmgr -d repmgr standby clone
    systemctl restart postgresql-9.6 &&  systemctl enable postgresql-9.6 # Run on root user
    repmgr standby register
    repmgr cluster show
<img src="./images/Screenshot 2021-11-05 235637.png" />  

# Slave-2
# Cấu hình file /etc/repmgr/9.6/repmgr.conf

    cp /etc/repmgr/9.6/repmgr.conf /etc/repmgr/9.6/repmgr.conf_org
    truncate -s 0 /etc/repmgr/9.6/repmgr.conf
    cat <<EOF>> /etc/repmgr/9.6/repmgr.conf
    cluster='pg_cluster'
    node_id=3
    node_name=node-3
    conninfo='host=10.0.0.13  user=repmgr dbname=repmgr connect_timeout=2'
    data_directory='/var/lib/pgsql/9.6/data'
    failover=automatic
    reconnect_attempts=3
    reconnect_interval=5
    promote_command='/usr/pgsql-9.6/bin/repmgr standby promote --log-to-file'
    follow_command='/usr/pgsql-9.6/bin/repmgr standby follow --log-to-file --upstream-node-id=%n'
    log_file='/var/log/repmgr/repmgrd-9.6.log'
    event_notifications='standby_promote'
    pg_bindir='/usr/pgsql-9.6/bin/'
    EOF
# Clone  and register standby database tu node master

    repmgr -h 10.0.0.11  -U repmgr -d repmgr standby clone
    systemctl restart postgresql-9.6 && systemctl enable postgresql-9.6 # Run on root user
    repmgr standby register
    repmgr cluster show
<img src="./images/Screenshot 2021-11-05 235927.png" />  

# High Availability
# Kịch bản node master down --> switch node-2 lên thành master. Node-3 replication từ new master node-2. Rejoin node-1 to cluster
# Bước 1 shutdown node-1 check cluster trên node-2 và node-3
<img src="./images/Screenshot 2021-11-06 000330.png" />  

<img src="./images/Screenshot 2021-11-06 000430.png" />  

# Bước 2: promote node-2 lên thành node master
    repmgr standby promote
<img src="./images/Screenshot 2021-11-06 001040.png" />  

# Bước 3 : Đúng trên node-2 và node-3 check status cluster

<img src="./images/Screenshot 2021-11-06 001120.png" />  

<img src="./images/Screenshot 2021-11-06 001146.png" />  

# Bước 4: Set node-3 follow standby node-2
    repmgr standby follow

<img src="./images/Screenshot 2021-11-06 001355.png" /> 

# Check lại trạng thái cluster
    repmgr cluster show
<img src="./images/Screenshot 2021-11-06 001512.png" /> 

# Bước 4: Start lại server node-1 và join lại vào cluster
# Đứng trên node-3 check lại trạng thái cluster, node-1 đã online tuy nhiên đã bị khai trừ ra khỏi cluster
    repmgr cluster show
<img src="./images/Screenshot 2021-11-06 004803.png" /> 

# Stop service postgresql trên node-1
    systemctl stop postgresql-9.6

# Rejoin cluster
    repmgr node rejoin -d 'host=node-2 dbname=repmgr user=repmgr' --force-rewind --config-files=postgresql.local.conf,postgresql.conf --verbose
    repmgr standby follow
<img src="./images/Screenshot 2021-11-06 005951.png" /> 
<img src="./images/Screenshot 2021-11-06 010837.png" />

# Cài đặt auto switch master
# Kiểm tra trạng thái cluster và repmgr service
    repmgr cluster show
    repmgr service status
<img src="./images/Screenshot 2021-11-06 102357.png" />

# Start repmgr-96 trên tất cả các node:    
    systemctl restart repmgr-96.service && systemctl enable repmgr-96.service
# Check status repmgr
     repmgr  service status
<img src="./images/Screenshot 2021-11-06 102944.png" />

# Check trạng thái cluster: master: node-1, standby: node-2, node-3
    repmgr  cluster show
<img src="./images/Screenshot 2021-11-06 100705.png" />

# Tiến hành stop service postgresql-9.6 trên service node-1 và check lại trạng thái cluster
    systemctl stop postgresql-9.6
    repmgr  cluster show
# Node-2 đã được switch thành master, node-3 đã được upstream theo node-2
<img src="./images/Screenshot 2021-11-06 103107.png" />    

# Tiến hành stop service postgresql-9.6 trên service node-2 và check lại trạng thái cluster
    systemctl stop postgresql-9.6
    repmgr  cluster show
# Node-3 đã được switch thành master
<img src="./images/Screenshot 2021-11-06 103247.png" />    

# Start lại server node-1, node-2 và join lại vào cluster

# Stop service postgresql trên node-1, node-2
    systemctl stop postgresql-9.6

# Rejoin cluster
     repmgr node rejoin -d 'host=node-3 dbname=repmgr user=repmgr' --force-rewind --config-files=postgresql.local.conf,postgresql.conf --verbose
<img src="./images/Screenshot 2021-11-06 103701.png" />  
<img src="./images/Screenshot 2021-11-06 103801.png" /> 

# Cài đặt Haproxy loadblance
# Config trên tất cả các node-pg
    yum install xinetd -y
# Tạo file confg xinetd check service postgres
    touch /etc/xinetd.d/postgreschk
    cat <<EOF>> /etc/xinetd.d/postgreschk
    # default: on
    # description: postgreschk
    service postgreschk
    {
            flags           = REUSE
            socket_type     = stream
            port            = 9201
            wait            = no
            user            = root
            server          = /usr/local/sbin/postgreschk
            log_on_failure  += USERID
            disable         = no
            #only_from       = 0.0.0.0/0
            only_from       = 0.0.0.0/0
            per_source      = UNLIMITED
    }
    EOF
# Add service check port postgres
        
# Tạo script check service postgres
    touch /usr/local/sbin/postgreschk
    chmod +x /usr/local/sbin/postgreschk
    cat <<EOF>> /usr/local/sbin/postgreschk
    #!/bin/bash
    #
    # This script checks if a PostgreSQL server is healthy running on localhost. It will
    # return:
    # "HTTP/1.x 200 OK\r" (if postgres is running smoothly)
    # - OR -
    # "HTTP/1.x 500 Internal Server Error\r" (else)
    #
    # The purpose of this script is make haproxy capable of monitoring PostgreSQL properly
    #

    export PGHOST='10.0.0.13' # Doi IP cua tung node
    export PGUSER='repmgr'
    export PGPASSWORD='1'
    export PGPORT='5432'
    export PGDATABASE='repmgr'
    export PGCONNECT_TIMEOUT=10

    FORCE_FAIL="/dev/shm/proxyoff"

    SLAVE_CHECK="SELECT pg_is_in_recovery()"
    WRITABLE_CHECK="SHOW transaction_read_only"

    return_ok()
    {
        echo -e "HTTP/1.1 200 OK\r\n"
        echo -e "Content-Type: text/html\r\n"
        if [ "$1x" == "masterx" ]; then
            echo -e "Content-Length: 56\r\n"
            echo -e "\r\n"
            echo -e "<html><body>PostgreSQL master is running.</body></html>\r\n"
        elif [ "$1x" == "slavex" ]; then
            echo -e "Content-Length: 55\r\n"
            echo -e "\r\n"
            echo -e "<html><body>PostgreSQL slave is running.</body></html>\r\n"
        else
            echo -e "Content-Length: 49\r\n"
            echo -e "\r\n"
            echo -e "<html><body>PostgreSQL is running.</body></html>\r\n"
        fi
        echo -e "\r\n"

        unset PGUSER
        unset PGPASSWORD
        exit 0
    }

    return_fail()
    {
        echo -e "HTTP/1.1 503 Service Unavailable\r\n"
        echo -e "Content-Type: text/html\r\n"
        echo -e "Content-Length: 48\r\n"
        echo -e "\r\n"
        echo -e "<html><body>PostgreSQL is *down*.</body></html>\r\n"
        echo -e "\r\n"

        unset PGUSER
        unset PGPASSWORD
        exit 1
    }

    if [ -f "$FORCE_FAIL" ]; then
        return_fail;
    fi

    # check if in recovery mode (that means it is a 'slave')
    SLAVE=$(/usr/pgsql-9.6/bin/psql -qt -c "$SLAVE_CHECK" 2>/dev/null)
    if [ $? -ne 0 ]; then
        return_fail;
    elif echo $SLAVE | egrep -i "(t|true|on|1)" 2>/dev/null >/dev/null; then
        return_ok "slave"
    fi

    # check if writable (then we consider it as a 'master')
    READONLY=$(/usr/pgsql-9.6/bin/psql -qt -c "$WRITABLE_CHECK" 2>/dev/null)
    if [ $? -ne 0 ]; then
        return_fail;
    elif echo $READONLY | egrep -i "(f|false|off|0)" 2>/dev/null >/dev/null; then
        return_ok "master"
    fi

    return_ok "none";
    EOF

# Config file /etc/haproxy/haproxy.cfg
    cp /etc/haproxy/haproxy.cfg /etc/haproxy/haproxy.cfg_org
    truncate -s 0 /etc/haproxy/haproxy.cfg
    cat <<EOF>> /etc/haproxy/haproxy.cfg
    global
        pidfile /var/run/haproxy.pid
        daemon
        user haproxy
        group haproxy
        stats socket /var/run/haproxy.socket user haproxy group haproxy mode 600 level admin
        node haproxy_haproxy-pg
        description haproxy server

        #* Performance Tuning
        maxconn 8192
        spread-checks 3
        quiet
    defaults
        #log    global
        mode    tcp
        option  dontlognull
        option tcp-smart-accept
        option tcp-smart-connect
        #option dontlog-normal
        retries 3
        option redispatch
        maxconn 8192
        timeout check   3500ms
        timeout queue   3500ms
        timeout connect 3500ms
        timeout client  10800s
        timeout server  10800s

    userlist STATSUSERS
        group admin users admin
        user admin insecure-password admin
        user stats insecure-password admin

    listen admin_page
        bind *:9600
        mode http
        stats enable
        stats refresh 60s
        stats uri /
        acl AuthOkay_ReadOnly http_auth(STATSUSERS)
        acl AuthOkay_Admin http_auth_group(STATSUSERS) admin
        stats http-request auth realm admin_page unless AuthOkay_ReadOnly
        #stats admin if AuthOkay_Admin

    listen  haproxy_haproxy-pg_5433_rw_rw
        bind *:5433
        mode tcp
        timeout client  10800s
        timeout server  10800s
        tcp-check connect port 9201
        tcp-check expect string master\ is\ running
        balance leastconn
        option tcp-check
    #   option allbackups
        default-server port 9201 inter 2s downinter 5s rise 3 fall 2 slowstart 60s maxconn 64 maxqueue 128 weight 100
        server 10.0.0.11 10.0.0.11:5432 check
        server 10.0.0.12 10.0.0.12:5432 check
        server 10.0.0.13 10.0.0.13:5432 check


    listen  haproxy_haproxy-pg_5434_ro
        bind *:5434
        mode tcp
        timeout client  10800s
        timeout server  10800s
        tcp-check connect port 9201
        tcp-check expect string is\ running
        balance leastconn
        option tcp-check
    #   option allbackups
        default-server port 9201 inter 2s downinter 5s rise 3 fall 2 slowstart 60s maxconn 64 maxqueue 128 weight 100
        server 10.0.0.11 10.0.0.11:5432 check
        server 10.0.0.12 10.0.0.12:5432 check
        server 10.0.0.13 10.0.0.13:5432 check
    EOF
    systemctl restart haproxy

# Check trạng thái backend trên dashboard: http://10.0.0.14:9600/ (User/pass: admin/admin)
<img src="./images/Screenshot 2021-11-06 163417.png" /> 
