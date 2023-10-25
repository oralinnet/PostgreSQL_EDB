### Configure Replica In EPAS(15)

#### Requrement 
- Two Or more EPAS cluster 
- Primary DB In Read Write Mode 
- Replica Server only EPAS Binary Installed
- Both Server sudo or root access 
- Ensure that Both servers can use ssh/scp passwordless communication
- Both Server Nedd Internet Access 
- Ensure Primary and replica DB is password base authentication active
- Primary : 192.168.5.240, Secondary: 192.168.5.241

#### Secondary / Replica Server 
- Login EDB websit (https://www.enterprisedb.com/accounts/profile) 
- Repo Access --> Access EDB Repos 2.0 
- Select Platform --> RHEL 8 --> Select software --> EDB Postgres Advanced Server (with version)

```sh
sudo curl -1sLf 'https://downloads.enterprisedb.com/zp579PrIC9a7kY4rQtxX63HAaXHtzeCA/enterprise/setup.rpm.sh' | sudo -E bash
sudo dnf -y install edb-as15-server
```

#### Primary Server 
```sh
sudo su - enterprisedb
psql edb

```