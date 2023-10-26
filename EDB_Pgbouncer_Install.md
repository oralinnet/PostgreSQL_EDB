### Install and configure EDB Pgbouncer on RHEL 8

#### Prerequisite
- Enable EDB subcription
- Internet Access 
- OS user with sudo privileges

#### Install Pgbouncer On RHEL 8
- Login EDB websit (https://www.enterprisedb.com/accounts/profile) 
- Repo Access --> Access EDB Repos 2.0 
- Select Platform --> RHEL 8 --> Select software --> EDB Postgres Advanced Server (with version)
```sh
sudo curl -1sLf 'https://downloads.enterprisedb.com/zp579PrIC9a7kY4rQtxX63HAaXHtzeCA/enterprise/setup.rpm.sh' | sudo -E bash
sudo dnf -y install edb-pgbouncer120
```
- Connect EDB pgbouncer to EPAS
```sh
## Go to configuration Directory
cd /etc/edb/pgbouncer1.20/
sudo vim userlist.txt
"enterprisedb" "edb"        ### enterprisedb user name of EPAS, edb is Database Name

### update configuration file information
sudo vim edb-pgbouncer-1.20.ini
[databases]     ## section
pgtest = host=192.168.5.242 port=5444 dbname=edb    ## pgtest con string name, host EPAS IP, port DB port, dbname EPAS db name.
listen_addr = *         # which ip allow to connect
listen_port = 6432      # pgbouncer port 
auth_file = /etc/edb/pgbouncer1.20/userlist.txt     ## username password file
pool_mode = session         ## pool mode
### Start Service and firewall configuration
sudo systemctl start edb-pgbouncer-1.20.service
sudo firewall-cmd --permanent --add-port=6432/tcp
sudo firewall-cmd --reload
sudo systemctl enable edb-pgbouncer-1.20.service    # Enable serive (optional)
```
- Ensure EDB Pgbouncer server IP address allow in EPAS pg_hba.conf file