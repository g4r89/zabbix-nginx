# zabbix-nginx

# prepare server
```bash
cat <<'EOF' > /etc/sysconfig/network-scripts/ifcfg-eth0
TYPE="Ethernet"
BOOTPROTO="static"
DEFROUTE="yes"
PEERDNS="yes"
PEERROUTES="yes"
IPV4_FAILURE_FATAL="no"
IPV6INIT="yes"
IPV6_AUTOCONF="yes"
IPV6_DEFROUTE="yes"
IPV6_PEERDNS="yes"
IPV6_PEERROUTES="yes"
IPV6_FAILURE_FATAL="no"
IPV6_ADDR_GEN_MODE="stable-privacy"
NAME="eth0"
UUID="f10bac81-5a8b-4c4d-bd48-2dbd212247c3"
DEVICE="eth0"
ONBOOT="yes"
IPADDR="10.0.14.251"
NETMASK="255.255.255.0"
GATEWAY="10.0.14.1"
DNS1="10.0.14.4"
EOF

systemctl restart network

echo 'zabbix.sk2.su' > /etc/hostname
yum install -y epel-release && yum -y update
sed -i.bak '/SELINUX/s/enforcing/permissive/' /etc/selinux/config
setenforce 0

systemctl disable --now firewalld
```

# install nginx
```bash
yum -y update
yum install -y nginx php-fpm

cat <<'EOF'> /etc/nginx/conf.d/zabbix.conf
server {
listen 80;

root /usr/share/zabbix;
access_log /var/log/nginx/zabbix.access.log;
server_name zabbix.site.ru;

location / {
index index.php index.html index.htm;
}

location ~ \.php$ {
fastcgi_pass unix:/var/run/php-fpm/php5-fpm.sock;
fastcgi_index index.php;
fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
include fastcgi_params;
fastcgi_param PHP_VALUE "
max_execution_time = 300
memory_limit = 128M
post_max_size = 16M
upload_max_filesize = 2M
max_input_time = 300
date.timezone = Europe/Moscow
always_populate_raw_post_data = -1
";
fastcgi_buffers 8 256k;
fastcgi_buffer_size 128k;
fastcgi_intercept_errors on;
fastcgi_busy_buffers_size 256k;
fastcgi_temp_file_write_size 256k;
}
}
EOF
```

# install mariadb
```bash
yum install -y mariadb-server
systemctl enable --now mariadb.service
/usr/bin/mysql_secure_installation
```

# install zabbix
```bash
cat <<'EOF'> /etc/yum.repos.d/zabbix.repo
[zabbix]
name=Zabbix Official Repository - $basearch
baseurl=http://repo.zabbix.com/zabbix/3.2/rhel/7/$basearch/
enabled=1
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-ZABBIX-A14FE591

[zabbix-non-supported]
name=Zabbix Official Repository non-supported - $basearch
baseurl=http://repo.zabbix.com/non-supported/rhel/7/$basearch/
enabled=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-ZABBIX
gpgcheck=1
EOF

yum install -y zabbix-server-mysql zabbix-web-mysql
mysql -uroot -p
create database zabbix character set utf8 collate utf8_bin;
grant all privileges on zabbix.* to zabbix@localhost identified by 'password';
flush privileges;
exit;

