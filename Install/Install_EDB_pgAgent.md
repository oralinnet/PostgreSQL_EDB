### Install and Configure EDB pgAgent in RHEL 8
#### Prerequisite
- Enable EDB subcription
- Internet Access 
- OS user with sudo privileges
- EPAS Configure with [password base authentication](https://github.com/oralinnet/PostgreSQL_EDB/blob/main/Install/Install_EPAS.md)
- Login EDB websit (https://www.enterprisedb.com/accounts/profile) 
- Repo Access --> Access EDB Repos 2.0 
- Select Platform --> RHEL 8 --> Select software --> EDB Postgres Advanced Server (with version)
```sh
sudo curl -1sLf 'https://downloads.enterprisedb.com/zp579PrIC9a7kY4rQtxX63HAaXHtzeCA/enterprise/setup.rpm.sh' | sudo -E bash
##### Install pgAgent in EPAS server 
 sudo dnf -y install edb-as15-pgagent
```
#### Configure pgAgent in EPAS server
- Change pgAgent parameter pgagent_15.conf file 
- Configure pgpass for bypass password which user is owner of EPAS

```sh
vim /etc/sysconfig/edb/pgagent15/edb-pgagent-15.conf

DBNAME=edb
DBUSER=enterprisedb
DBHOST=127.0.0.1
DBPORT=5444
PASSWORD=edb
LOGFILE=/var/log/pgagent_15.log

:x      ### save file 
```
- Create .pgpass file 

```sh
su - enterprisedb
vim .pgpass
######## syntax #####
hostname:port:database:username:password
######## syntax #####
127.0.0.1:5444:edb:enterprisedb:edb

```

#### Configure in EPAS 
- Create extension for pgagent
```sql
CREATE EXTENSION pgagent;
CREATE LANGUAGE plpgsql;
```

- Start pgAgent 
```sh
systemctl start edb-pgagent-15.service
systemctl enable edb-pgagent-15.service
```
- Now Reload PgAdmin 
- Check logfile 
```sh
vim /var/log/edb/pgagent15/pgagent.log
```
- Check service running or not 
```sh
systemctl status edb-pgagent-15.service
```
#### Create dump Backup from pgAgent 
- Create Directory for Backup and give proper permission to pgagent
```sh
mkdir /path/of/backup 
chown -R enterprisedb:enterprisedb /path/of/backup
chmo -R 775 /path/of/backup

#### pgagent batch file 
pg_dump -h 127.0.0.1 -d edb -f /path/of/backup/bkp-`date +%Y-%m-%d-%H-%M-%S`.sql -p 5444 -U enterprisedb
```
- After create pgagent job please check edb-pgagent-15.service running or not ensure it's running. 