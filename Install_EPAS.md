### Install EPAS(15) In RPM Base Linux Server 

####  Prerequisite 
- OS RHEL 8 With subscription 
- Enable EDB subcription
- Internet Access 
- OS user with sudo privileges
- Disable Selinux 

#### Install EPAS(15) In Default Location
- Login EDB websit (https://www.enterprisedb.com/accounts/profile) 
- Repo Access --> Access EDB Repos 2.0 
- Select Platform --> RHEL 8 --> Select software --> EDB Postgres Advanced Server (with version)
```sh
sudo curl -1sLf 'https://downloads.enterprisedb.com/zp579PrIC9a7kY4rQtxX63HAaXHtzeCA/enterprise/setup.rpm.sh' | sudo -E bash
sudo dnf -y install edb-as15-server
sudo PGSETUP_INITDB_OPTIONS="-E UTF-8" /usr/edb/as15/bin/edb-as-15-setup initdb        ## Database Default Location 
sudo systemctl start edb-as-15
sudo systemctl status edb-as-15
```
- Testing Database  
```sh
sudo su - enterprisedb
psql edb
### sql
ALTER ROLE enterprisedb IDENTIFIED BY edb;
CREATE DATABASE hr;
\c hr
CREATE TABLE public.dept (deptno numeric(2) NOT NULL CONSTRAINT dept_pk
PRIMARY KEY, dname varchar(14) CONSTRAINT dept_dname_uq UNIQUE, loc
varchar(13));
INSERT INTO dept VALUES (10,'ACCOUNTING','NEW YORK');
INSERT into dept VALUES (20,'RESEARCH','DALLAS');
SELECT * FROM dept;
```
#### Update EPAS PATH in enterprisedb user 
```sh
sudo su - enterprisedb
vim .bash_profile
# Postgres edb path
export PATH=$PATH:/usr/edb/as15/bin

### Now reload bash_profile or logout and login 
```

#### Password base authentication 
- First set or create passowd for database user

```sh
#### Login Database and alter or create password 
sudo su - enterprisedb
psql edb
### sql 
ALTER ROLE enterprisedb IDENTIFIED BY edb;
\q
###
```
- In pg_hba.conf change METHOD
```sh
sudo su - enterprisedb 
vim /var/lib/edb/as15/data/pg_hba.conf
    replace peer to scram-sha-256
    replace ident to scram-sha-256
:x          ## save file 
sudo systemctl restart edb-as-15            ### Restart database service or reload 
pg_ctl -D /var/lib/edb/as15/data reload     ### for reload 

```

- Check Password base authentication 
```sh
sudo su - enterprisedb 
psql -h /tmp -p 5444 -U enterprisedb -d edb     ### Now you need password for login database
```

#### Configure Database in Another Directory 
- First create directory for DATA
- Give proper permission in enterprisedb user 

```sh
sudo mkdir /mydb/ -p
sudo chown -R enterprisedb:enterprisedb /mydb
chmod -R 775 /mydb
sudo su - enterprisedb 
initdb -D /mydb -w                          ### install DB
pg_ctl -D /mydb -l /mydb/startlog start     ### start DB
pg_ctl -D /mydb/ status                     ### DB status
pg_ctl -D /mydb start                       ### Start DB
pg_ctl -D /mydb stop                        ### Stop DB

```

#### post installation task 
- Enable edb-as-15 service for auto start after reboot
- Open DB port in firewall 
- Accept Connection from out site in DB 

```sh
sudo systemctl enable edb-as-15
sudo firewall-cmd --permanent --add-port=5444/tcp
sudo firewall-cmd --reload 
sudo su - enterprisedb 
vim /var/lib/edb/as15/data/pg_hba.conf
host  all     all     192.168.5.241/32         scram-sha-256        ### cline ip address 
sudo systemctl restart edb-as-15            ### Restart database service or reload 
pg_ctl -D /var/lib/edb/as15/data reload     ### for reload 
```
