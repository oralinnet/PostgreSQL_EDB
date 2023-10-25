### Install EPAS(15) In RPM Base Linux Server 

#### Requrement 
- OS RHEL 8 With subscription 
- Enable EDB subcription
- Internet Access 
- OS user with sudo privileges

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

#### Password base auth 

```sh
#### Login Database and alter or create password 
sudo su - enterprisedb
psql edb

```sql
ALTER ROLE enterprisedb IDENTIFIED BY edb;
\q
```

```

