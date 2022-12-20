# How to Install Zabbix 6.0 / 6.2 on Ubuntu 22.04 / 20.04 [Step-by-Step]

## Contents

* Step 1: Install Zabbix server, frontend, and agent
* Step 2: Configure database
* Step 3: Configure firewall
* Step 4: Start Zabbix server and agent processes
* Step 5: Configure Zabbix frontend
* Step 6: Login to frontend using Zabbix default login credentials
* Step 7: Optimizing Zabbix Server (optional)
* Step 8: Optimizing MySQL / MariaDB database (optional)
* Step 9: Create MySQL partitions on History and Events tables
* Step 10: How to manage Zabbix / MySQL / Apache service
* Step 11: Upgrade between minor versions

# Step 1: Install Zabbix server, frontend, and agent
Zabbix 6.0 LTS version (supported until February, 2027)
```
wget https://repo.zabbix.com/zabbix/6.0/ubuntu/pool/main/z/zabbix-release/zabbix-release_6.0-4+ubuntu$(lsb_release -rs)_all.deb
```
```
sudo dpkg -i zabbix-release_6.0-4+ubuntu$(lsb_release -rs)_all.deb
sudo apt update
sudo apt -y install zabbix-server-mysql zabbix-frontend-php zabbix-apache-conf zabbix-sql-scripts zabbix-agent
```

                        OR

Zabbix 6.2 standard version (supported until January, 2023)
```
wget https://repo.zabbix.com/zabbix/6.2/ubuntu/pool/main/z/zabbix-release/zabbix-release_6.2-2+ubuntu$(lsb_release -rs)_all.deb
```
```
sudo dpkg -i zabbix-release_6.2-2+ubuntu$(lsb_release -rs)_all.deb
sudo apt update
sudo apt -y install zabbix-server-mysql zabbix-frontend-php zabbix-apache-conf zabbix-sql-scripts zabbix-agent
```

# Step 2: Configure database

In this installation, I will use password rootDBpass as root password and zabbixDBpass as Zabbix password for DB. Consider changing your password for security reasons.

```diff
+ A. Install MariaDB 10.6.
```


In your terminal, use the following command to install MariaDB 10.6.

```
sudo apt install software-properties-common -y
curl -LsS -O https://downloads.mariadb.com/MariaDB/mariadb_repo_setup
sudo bash mariadb_repo_setup --mariadb-server-version=10.6
sudo apt update
sudo apt -y install mariadb-common mariadb-server-10.6 mariadb-client-10.6
```
Once the installation is complete, start the MariaDB service and enable it to start on boot using the following commands:
```
sudo systemctl start mariadb
sudo systemctl enable mariadb
```
```diff
+ B. Reset root password for database.
```


Secure MySQL/MariaDB by changing the default password for MySQL root:
```
sudo mysql_secure_installation

Enter current password for root (enter for none): Press Enter
Switch to unix_socket authentication [Y/n] y
Change the root password? [Y/n] y
New password: <Enter root DB password>
Re-enter new password: <Repeat root DB password>
Remove anonymous users? [Y/n]: Y
Disallow root login remotely? [Y/n]: Y
Remove test database and access to it? [Y/n]:  Y
Reload privilege tables now? [Y/n]:  Y
```
```diff
+ C. Create database.
```
```
sudo mysql -uroot -p'rootDBpass' -e "create database zabbix character set utf8mb4 collate utf8mb4_bin;"
sudo mysql -uroot -p'rootDBpass' -e "grant all privileges on zabbix.* to zabbix@localhost identified by 'zabbixDBpass';"
```
```diff
+ D. Import initial schema and data.
```
Import database shema for Zabbix server (could last up to 5 minutes):
```
sudo zcat /usr/share/zabbix-sql-scripts/mysql/server.sql.gz | mysql --default-character-set=utf8mb4 -uzabbix -p'zabbixDBpass' zabbix
```
```diff
+ E. Enter database password in Zabbix configuration file.
```

Open zabbix_server.conf file with command:
```
sudo nano /etc/zabbix/zabbix_server.conf
```
and add database password in this format anywhere in file:
```
DBPassword=zabbixDBpass
Save and exit file (ctrl+x, followed by y and enter).
```

# Step 3: Configure firewall

If you have a UFW firewall installed on Ubuntu, use these commands to open TCP ports: 10050 (agent), 10051 (server), and 80 (frontend), and port:22 (ssh) :
```
ufw allow 10050/tcp
ufw allow 10051/tcp
ufw allow 80/tcp
ufw allow 22/tcp
ufw reload
```

# Step 4: Start Zabbix server and agent processes
```
sudo systemctl restart zabbix-server zabbix-agent
sudo systemctl enable zabbix-server zabbix-agent
Step 5: Configure Zabbix frontend
```
```diff
+ A. Configure PHP for Zabbix frontend
```
```
Edit file /etc/zabbix/apache.conf:
```
```
sudo nano /etc/zabbix/apache.conf
```
Uncomment 2 lines in apache.conf that starts with “# php_value date.timezone Europe/Riga” by removing symbol # and set the right timezone for your country, for example:

php_value date.timezone Europe/Amsterdam
Save and exit file (ctrl+x, followed by y and enter)

```diff
+ B. Restart Apache web server and make it start at system boot
```

```
sudo systemctl restart apache2
sudo systemctl enable apache2
```
```diff
+ C. Configure web frontend start at system boot
```


Connect to your newly installed Zabbix frontend using URL “http://server_ip_or_dns_name/zabbix” to initiate the Zabbix installation wizard.

In my case, that URL would be “http://192.168.1.161/zabbix” because I have installed Zabbix on the server with IP address 192.168.1.161 (you can find the IP address of your server by typing “ip a” command in the terminal).

Basically, in this wizard you only need to enter a password for Zabbix DB user and just click “Next step” for everything else. In this guide, I have used a zabbixDBpass as a database password, but if you set something else, be sure to enter the correct password when prompted by the wizard.

# Step 6: Login to frontend using Zabbix default login credentials

Use Zabbix default admin username “Admin” and password “zabbix” (without quotes) to login to Zabbix frontend at URL “http://server_ip_or_dns_name/zabbix” via your browser.

# Step 7: Optimizing Zabbix Server (optional)

Don’t bother with this optimization if you are monitoring a small number of devices, but if you are planning to monitor a large number of devices then continue with this step.

Open “zabbix_server.conf” file with command: “sudo nano /etc/zabbix/zabbix_server.conf” and add this configuration anywhere in file:
```
StartPollers=100
StartPollersUnreachable=50
StartPingers=50
StartTrappers=10
StartDiscoverers=15
StartPreprocessors=15
StartHTTPPollers=5
StartAlerters=5
StartTimers=2
StartEscalators=2
CacheSize=128M
HistoryCacheSize=64M
HistoryIndexCacheSize=32M
TrendCacheSize=32M
ValueCacheSize=256M
```
Save and exit file (ctrl+x, followed by y and enter).

This is not a perfect configuration, keep in mind that you can optimize it even more. Let’s say if you don’t use ICMP checks then set the “StartPingers” parameter to 1 or if you don’t use active agents then set “StartTrappers” to 1 and so on. You can find out more about the parameters supported in a Zabbix server configuration file in the official documentation.

If you try to start the Zabbix server you may receive an error “[Z3001] connection to database 'Zabbix' failed: [1040] Too many connections” in the log “/var/log/zabbix/zabbix_server.log” because we are using more Zabbix server processes than MySQL can handle. We need to increase the maximum permitted number of simultaneous client connections and optimize MySQL – so move to the next step.

# Step 8: Optimizing MySQL / MariaDB database (optional)

```diff
+ A. Create custom MySQL configuration file
```


Create file “10_my_tweaks.cnf" with “sudo nano /etc/mysql/mariadb.conf.d/10_my_tweaks.cnf” and paste this configuration:
```
[mysqld]
max_connections                = 404
innodb_buffer_pool_size        = 800M

innodb-log-file-size           = 128M
innodb-log-buffer-size         = 128M
innodb-file-per-table          = 1
innodb_buffer_pool_instances   = 8
innodb_old_blocks_time         = 1000
innodb_stats_on_metadata       = off
innodb-flush-method            = O_DIRECT
innodb-log-files-in-group      = 2
innodb-flush-log-at-trx-commit = 2

tmp-table-size                 = 96M
max-heap-table-size            = 96M
open_files_limit               = 65535
max_connect_errors             = 1000000
connect_timeout                = 60
wait_timeout                   = 28800
```
Save and exit the file (ctrl+x, followed by y and enter) and set the correct file permission:
```
sudo chown mysql:mysql /etc/mysql/mariadb.conf.d/10_my_tweaks.cnf
sudo chmod 644 /etc/mysql/mariadb.conf.d/10_my_tweaks.cnf
```
Two things to remember!

Configuration parameter max_connections must be larger than the total number of all Zabbix proxy processes plus 150. You can use the command below to automatically check the number of Zabbix processes and add 150 to that number:

root@ubuntu:~ $ egrep "^Start.+=[0-9]"  /etc/zabbix/zabbix_server.conf  |  awk -F "=" '{s+=$2} END {print s+150}'
 404
The second most important parameter is innodb_buffer_pool_size, which determines how much memory can MySQL get for caching InnoDB tables and index data. You should set that parameter to 70% of system memory if only database is installed on server.

However, in this case, we are sharing a server with Zabbix and Apache processes so you should set innodb_buffer_pool_size to 40% of total system memory. That would be 800 MB because my Ubuntu server has 2 GB RAM.

I didn’t have any problems with memory, but if your Zabbix proxy crashes because of lack of memory, reduce “innodb_buffer_pool_size” and restart MySQL server.

Note that if you follow this configuration, you will receive “Too many processes on the Zabbix server” alarm in Zabbix frontend due to the new Zabbix configuration. It is safe to increase the trigger threshold or turn off that alarm (select “Problems” tab → left click on the alarm → select “Configuration” → remove the check from “Enabled” → hit the “Update” button)

```diff
+ B. Restart Zabbix Server and MySQL service
```

Stop and start the services in the same order as below:
```
sudo systemctl stop zabbix-server
sudo systemctl stop mysql
sudo systemctl start mysql
sudo systemctl start zabbix-server
```

# Step 9: Create MySQL partitions on History and Events tables

Zabbix’s housekeeping process is responsible for deleting old trend and history data. Removing old data from the database using SQL delete query can negatively impact database performance. Many of us have received that annoying alarm “Zabbix housekeeper processes more than 75% busy” because of that.

That problem can be easily solved with the database partitioning. Partitioning creates tables for each hour or day and drops them when they are not needed anymore. SQL DROP is way more efficient than the DELETE statement.

You can partition MySQL tables in 5 minutes using this simple guide.

# Step 10: How to manage Zabbix / MySQL / Apache service

Sometimes you will need to check or restart Zabbix, MySQL or Apache service – use commands below to do that.
```
Zabbix Server
sudo systemctl <status/restart/start/stop> zabbix-server

MySQL Server
sudo systemctl <status/restart/start/stop> mysql

Apache Server
sudo systemctl <status/restart/start/stop> apache2

Zabbix Agent
sudo systemctl <status/restart/start/stop> zabbix-agent
```
# Step 11: Upgrade between minor versions

I wrote about these upgrade procedures in my post about Zabbix upgrade. Zabbix’s team releases new minor versions at least once a month. The main purpose of minor upgrades is to fix bugs (hotfix) and sometimes even bring new functionality. Therefore, try to do a minor upgrade of Zabbix at least once a month.

There is no need for backups when doing a minor upgrade, they are completely safe. With this command you can easily upgrade smaller versions of 6.0.x (for example, from 6.0.1 to 6.0.3):
```
sudo apt install --only-upgrade 'zabbix*'
And restart Zabbix server afterward:
```
```
sudo systemctl restart zabbix-server
```

