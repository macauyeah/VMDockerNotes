前述我們一直在介紹docker cluster，但docker也不是萬能的。某些依賴HDD的程式，而且檔案權限相對有要求的程式，例如：資料庫，用docker去接入共享的HDD (mout share storage)，反而更麻煩。一來需要程式本身支援，二來要修改官方docker的初始化程序，過程相關折騰。所以有這方法需求的，都可以先考慮原本的VM做成Cluster。

本文就介紹一下傳統的Mariadb 做成Cluster的方式。老實講，Mariadb 官方手冊可能因為要適配各個不同的OS品牌，並沒有提供一個平台完整的安裝流程。最後筆者也是轉向一些非官方的網絡教學，才成功設定。

# Galera 4 (Mariadb cluster) on ubuntu 24

https://www.linode.com/docs/guides/how-to-set-up-mariadb-galera-clusters-on-ubuntu-2204/

筆者參考上述文章，配合自己測試的結果，以下簡介一下在Ubuntu 24.04的安裝過程

準備3台VM，假設它們的IP為 192.168.0.2, 192.168.0.3, 192.168.0.4 ，確保它們之間的網路可以互通，每一台機都執行以下的安裝script

```bash
sudo apt-get install mariadb-server mariadb-client galera-4
sudo mysql_secure_installation
```

## NODE1 192.168.0.2

修改/etc/mysql/mariadb.conf.d/60-galera.cnf,  留意 wsrep_node_address, wsrep_node_name 部份，要與本機相同。

```
[galera]
# Mandatory settings
wsrep_on                 = ON
wsrep_provider           = /usr/lib/galera/libgalera_smm.so
wsrep_cluster_name       = "pocdbcluster"
wsrep_cluster_address    = "gcomm://192.168.0.2,192.168.0.3,192.168.0.4"
binlog_format            = row
default_storage_engine   = InnoDB
innodb_autoinc_lock_mode = 2

# Allow server to accept connections on all interfaces.
bind-address = 0.0.0.0
wsrep_node_address="192.168.0.2"
wsrep_node_name="pocdbnode1"
```

設定好後，我們可以關掉mariadb，經galera 新起cluster的方式叫起它，然後經sql 在內部查看現時成功有加到cluster的機器數量。

```bash
sudo systemctl stop mariadb
sudo galera_new_cluster  # will start up mariadb
mysql -u root -p -e "SHOW STATUS LIKE 'wsrep_cluster_size'"
```

應該要看到數量為1

## NODE2 192.168.0.3

在node2，跟進述一樣，修改 /etc/mysql/mariadb.conf.d/60-galera.cnf，記得wsrep_node_address, wsrep_node_name要換成新機的值。

```
[galera]
# Mandatory settings
wsrep_on                 = ON
wsrep_provider           = /usr/lib/galera/libgalera_smm.so
wsrep_cluster_name       = "pocdbcluster"
wsrep_cluster_address    = "gcomm://192.168.0.2,192.168.0.3,192.168.0.4"
binlog_format            = row
default_storage_engine   = InnoDB
innodb_autoinc_lock_mode = 2

# Allow server to accept connections on all interfaces.
bind-address = 0.0.0.0
wsrep_node_address="192.168.0.3"
wsrep_node_name="pocdbnode2"
```

設定好後，就重啟mariadb，順便看看現時成功有加到cluster的機器數量。應該要看到數量為2

```bash
sudo systemctl restart mariadb
mysql -u root -p -e "SHOW STATUS LIKE 'wsrep_cluster_size'"
```

## NODE3 192.168.0.4

在node3，跟進述一樣，設定值筆者就省略了。我們可以在測試一次真實的改動，是否可以同步到其他node。

我們先試在node3加入新的資料庫test1，然後在node2查看是否存在。

```bash
# node3
mysql -u root -p -e "create database test1;"
# node2
mysql -u root -p -e "show databases;"
```

node2應該是可以找到test1的，不然就要經過查wsrep_cluster_size看看node3是否成功接入。

然後我們再在node2試試修改root的密碼，看看會不會同步到其他node。

```bash
# node2
mysql -u root -p
# in sql
MariaDB [(none)]> SET PASSWORD for 'root'@'localhost' = PASSWORD('YOUR_NEW_PW');
# node1
mysql -u root -p
```

最後node1, node3都需要使用新的password才能登入。

當一切都如預期，你的Mariadb Galera cluster就成功了。