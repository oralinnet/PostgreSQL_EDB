### Install EDB PEM ON RHEL 8

#### Prerequisite
- EDB v15 is installed on server with password base DB authentication
- Enable EDB subcription
- Internet Access 
- OS user with sudo privileges

#### Install and Configure PEM on RHEL 8

#### Install Pgbouncer On RHEL 8
- Login EDB websit (https://www.enterprisedb.com/accounts/profile) 
- Repo Access --> Access EDB Repos 2.0 
- Select Platform --> RHEL 8 --> Select software --> EDB Postgres Advanced Server (with version)
```sh
sudo curl -1sLf 'https://downloads.enterprisedb.com/zp579PrIC9a7kY4rQtxX63HAaXHtzeCA/enterprise/setup.rpm.sh' | sudo -E bash
sudo dnf -y install edb-as15-server
sudo PGSETUP_INITDB_OPTIONS="-E UTF-8" /usr/edb/as15/bin/edb-as-15-setup initdb
sudo systemctl start edb-as-15
sudo su - enterprisedb
psql edb
ALTER ROLE enterprisedb IDENTIFIED BY edb;
```
- Install required package and PEM
```sh
sudo dnf -y install edb-as15-server-sslutils
sudo dnf -y install edb-pem
```
- Configure PEM 
```sh
sudo /usr/edb/pem/bin/configure-pem-server.sh

Install type: 1:Web Services and Database, 2:Web Services 3: Database [ ] :1
Enter local database server installation path (i.e. /usr/edb/as12 , or /usr/pgsql-12, etc.) [ ] :/usr/edb/as15/
Enter database super user name [ ] :enterprisedb        --> Install on this server
Enter database server port number [ ] :5444             --> This server port 
Enter database super user password [ ] :                --> This server
Please enter CIDR formatted network address range that agents will connect to the server from, to be added to the server's pg_hba.conf file. For example, 192.168.1.0/24 [ 0.0.0.0/0 ] :192.168.5.0/24  --> which ip can connect with PEM
Enter database systemd unit file or init script name (i.e. edb-as-12 or postgresql-12, etc.) [ ] :edb-as-15
Please specify agent certificate path (Script will attempt to create this directory, if it does not exists) [ /root/.pem/ ] :       --> Enter 
```
- Post Installation
```sh
sudo firewall-cmd --permanent --add-port=8443/tcp
sudo firewall-cmd --reload

# httpd server enable and start after reboot 
sudo systemctl enable httpd
# PEM Server Interface 
https://192.168.5.242:8443/pem

## login user name and password 
user: enterprisedb          --> DB user name, That install On PEM Server
pass: edb                   --> DB password 
```