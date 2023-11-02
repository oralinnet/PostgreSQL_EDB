### Configure streaming replication In EPAS(15)

####  Prerequisite
- Two Or more EPAS cluster 
- Primary DB In Read Write Mode 
- Replica Server only EPAS Binary Installed
- Both Server sudo or root access 
- Ensure that Both servers can use ssh/scp passwordless communication
- Both Server Need Internet Access 
- Ensure Primary and replica DB is password base authentication active
- Primary : 192.168.5.240, Secondary: 192.168.5.241
- OS RHEL 8
- Used Package : rsync vim net-tools 

#### Secondary / Replica Server 
- Login EDB websit (https://www.enterprisedb.com/accounts/profile) 
- Repo Access --> Access EDB Repos 2.0 
- Select Platform --> RHEL 8 --> Select software --> EDB Postgres Advanced Server (with version)
- Install EPAS Binary 

```sh
sudo curl -1sLf 'https://downloads.enterprisedb.com/zp579PrIC9a7kY4rQtxX63HAaXHtzeCA/enterprise/setup.rpm.sh' | sudo -E bash
sudo dnf -y install edb-as15-server
```

#### Primary Server Configure
- Enable Archive mode
- Configure postgresql.conf file 
- Change archive log location (optional)
```sh
sudo su - enterprisedb
psql edb
show archive_mode;              # Check archive mode 
\q      ## exit from DB 
mkdir /var/lib/edb/as15/ARCHIVELOG  -p
chown -R enterprisedb:enterprisedb /var/lib/edb/as15/ARCHIVELOG         ## no need if you already login enterprisedb user
### Enable archive mode 
sudo su - enterprisedb
psql edb
show config_file;           ### postgresql.conf file location
vim /var/lib/edb/as15/data/postgresql.conf
promote_trigger_file = '/tmp/trigger_file.trg'          ### For swithover 
archive_mode = on               ### archive mode on 
archive_command = 'test ! -f enterprisedb@192.168.5.241:/var/lib/edb/as15/ARCHIVELOG/%f && rsync -a %p enterprisedb@192.168.5.241:/var/lib/edb/as15/ARCHIVELOG/%f'      ### archive log sent replica server 
wal_level = replica     
max_wal_senders = 3
max_replication_slots = 4
:x      ### save file 
sudo systemctl restart edb-as-15            ### Restart database service or reload 
psql edb            #### login database 
show archive_mode;  #### Check archive mode 
CREATE ROLE replication WITH REPLICATION PASSWORD 'hello' LOGIN;        #### Create role for replication 
\q
vim /var/lib/edb/as15/data/postgresql.conf
listen_addresses = '*'		#### in production must use Production server ip 
#### Accept replica server IP
vim /var/lib/edb/as15/data/pg_hba.conf
# IPv4 local connections:
host    all             all             0.0.0.0/0              scram-sha-256
host  replication     replication     192.168.5.241/32         scram-sha-256

systemctl restart edb-as-15.service
```

- Configure ssh keybase authentication on enterprisedb user both server
```sh
### Primary site 
sudo passwd enterprisedb 
su - enterprisedb 
ssh-agent
ssh-copy-id enterprisedb@192.168.5.241
ssh enterprisedb@192.168.5.241
### Replica site (If you have swithover plan )
sudo passwd enterprisedb 
su - enterprisedb 
ssh-agent
ssh-copy-id enterprisedb@192.168.5.240
ssh enterprisedb@192.168.5.240
```

-- configure firewall in both server 
```sh
sudo firewall-cmd --permanent --add-port=5444/tcp
sudo firewall-cmd --reload 
```
### Replica server configuration
- Stop edb-as-15 service if its running most importance 
- Backup data directory if you have 
```sh
sudo systemctl stop edb-as-15
sudo su - enterprisedb
mkdir /var/lib/edb/as15/ARCHIVELOG  -p
pg_basebackup -h 192.168.5.240 -U enterprisedb -D /var/lib/edb/as15/data -U replication -v -P --wal-method=stream --write-recovery-conf
### 192.168.5.240   primary server ip, enterprisedb Database user, /var/lib/edb/as15/data directory location replica server, replication DB user 
touch /var/lib/edb/as15/data/standby.signal
sudo systemctl restart edb-as-15.service        ### restart service
```
#### Replication mode change 
- By default replication mode is async
```sh
#### Primary Server
su - enterprisedb 
psql edb 
select usename, application_name, client_addr, state, sync_priority, sync_state from pg_stat_replication;       ## check mode 

vim /var/lib/edb/as15/data/postgresql.conf

synchronous_commit = remote_apply
synchronous_standby_names = 'replica1'      ### replica1 is replica server cluster name 

### Secondary server
vim /var/lib/edb/as15/data/postgresql.conf

cluster_name = 'replica1'			### cluser name of replaca server
hot_standby = on
primary_conninfo = 'host=192.168.5.240 port=5444 user=replication'

#### Restart Both server service and check 
### primary site 
su - enterprisedb 
psql edb 
select usename, application_name, client_addr, state, sync_priority, sync_state from pg_stat_replication;
```

#### Post installation 
- Check replication 
- Check wal file move to replica server 
```sh
select pg_switch_wal();         ### switch wal file
```
- NB
```
If you change your server IP address maybe your replication is not working. Please reconfigure replication again. 
```