# PostgreSQL_EDB

#### Lab Setup Requrement 
- Centos 7 with 2 GB RAM 40 GB Free space 
- Latest Version of EDB PostgreSQL Advance Server must be install
- OS super user access 
- EDB credentials for EDB Repos 2.0
- Internet access 

#### Requred Package 
```sh
yum -y install edb-as15-server
yum -y install edb-as15-server-contrib
yum -y install edb-as15-server-sslutils
yum -y install edb-pem
yum -y install edb-as15-server-sqlprofiler
yum -y install edb-as15-server-indexadvisor
yum -y install edb-as15-server-edb_wait_states
yum -y install edb-pgbouncen119
yum -y install edb-pgpool44
yum -y install edb-as15-pgpool44-extensions
```
### Install 
```sh
useradd enterprisedb
passwd enterprisedb
curl -1sLf 'https://downloads.enterprisedb.com/zp579PrIC9a7kY4rQtxX63HAaXHtzeCA/enterprise/setup.rpm.sh' | sudo -E bash
yum -y install edb-as15-server
sudo PGSETUP_INITDB_OPTIONS="-E UTF-8" /usr/edb/as15/bin/edb-as-15-setup initdb
sudo systemctl start edb-as-15
systemctl enable edb-as-15.service
```


