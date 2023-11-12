### EFM Cluster Install and configure on RHEL 8

#### Prerequisite
- EDB v15 is installed in the primary and replica server 
- Replication is ongoing 
- All the servers are reachable from each other 
- Ports are open  
- Make sure Java is installed (version 1.8 or above open jdk)
- Make sure all server has internet access 
- Make sure all server has EDB repo 
- Ensure you have VIP (virtual IP )
- witness server no need EPAS 
- Ensure Primary DB and Standby DB password base authentication 
- Adjust firewall to allow EFM agents to communicate on ports 7800 and 7809 (depend on your EFM admin port)

- In my case I used Three Servers 
```t
Master Node (node1) - 192.168.5.240     -- Primary DB
Standby Node (node2)- 192.168.5.240     -- Standby DB
Witness Node (node3) - 192.168.5.240
VIP - 192.168.5.243
```
- Login EDB websit (https://www.enterprisedb.com/accounts/profile) 
- Repo Access --> Access EDB Repos 2.0 
- Select Platform --> RHEL 8 --> Select software --> EDB Postgres Advanced Server (with version)
- Install EPAS Binary 

```sh
sudo curl -1sLf 'https://downloads.enterprisedb.com/zp579PrIC9a7kY4rQtxX63HAaXHtzeCA/enterprise/setup.rpm.sh' | sudo -E bash
```
- Install Java on all servers 
```sh
sudo dnf install java -y
java -version
```
- Update pg_hba.conf file on both primary and standby server with the information of the replication and witness servers
- Reload pg_hba.conf file on both primary and standby server
- withness server no need EPAS only EFM install there 
```t
host    all             all            192.168.5.240/32           scram-sha-256
host    all             all            192.168.5.241/32           scram-sha-256
host    all             all            192.168.5.242/32           scram-sha-256
```
- check reload status on both primary and standby server 
```sql
select pg_reload_conf();
```
- Install EFM binary in all servers Primary ,Standby and witness server 
```sh
sudo dnf -y install edb-efm47
```
#### Configure EFM on Primary DB Server 
- Create super user for EFM
```sh
su - enterprisedb 
psql edb
create user efm with password 'hello' superuser ;
sudo su - efm
psql edb
```
- Configure EFM configuration file 
- Go to EFM Directory and copy of the template before modifying the file contents and change the ownership
```sh
cd /etc/edb/efm-4.7/
cp efm.properties.in  efm.properties
cp efm.nodes.in  efm.nodes
sudo chown efm:efm efm.properties
sudo chown efm:efm efm.nodes
```
- Encrypt password of efm DB user 
```sh
/usr/edb/efm-4.7/bin/efm encrypt efm
## db.password.encrypted=cn3zba41e84d67787834c16bdfeae3f5   
```
- Make changes to efm.properties file 

```sh
vim efm.properties

db.user=efm
db.password.encrypted=c0c2f3d8c85ad2c7d59ee7fceb27bddd
db.port=5444
db.database=edb
db.service.owner=enterprisedb
db.service.name= edb-as-15
db.bin=/usr/edb/as15/bin/
db.data.dir=/var/lib/edb/as15/data/
db.config.dir=/var/lib/edb/as15/data/
bind.address=192.168.5.240:7800
is.witness=false
user.email=rakibul@gmail.com
ping.server.ip=192.168.5.1		-- ip muse be alive 	
auto.allow.hosts=true
virtual.ip=192.168.5.243				
virtual.ip.interface=eth0
virtual.ip.prefix=24
virtual.ip.single=true

## NB: user.email can't blank 
```
- Add nodes entries in efm.nodes 
```sh
vim efm.nodes

192.168.5.240:7800
192.168.5.241:7800
192.168.5.242:7800
```
- Start Service and configure firewall
```sh
sudo systemctl start edb-efm-4.7.service
sudo firewall-cmd --permanent --add-port={7800,7809}/tcp        ## 7809 is admin port 
sudo firewall-cmd --reload

```
- Check EFM Status 
```sh
/usr/edb/efm-4.7/bin/efm cluster-status efm
```
- Path export on user .bash_profile and reload it 
```sh
export PATH=$PATH:/usr/edb/efm-4.7/bin
```
- allow node 
```sh
/usr/edb/efm-4.7/bin/efm allow-node  efm 192.168.5.241
```

#### EFM Configuration on Standby Node 
- copy efm configuration file primary to standby node 
```sh
### primary node 
sudo scp efm.nodes efm.properties root@192.168.5.241:/etc/edb/efm-4.7
### standby node 
sudo chown efm:efm /etc/edb/efm-4.7/efm.properties
sudo chown efm:efm /etc/edb/efm-4.7/efm.nodes
```
- Keep everything as it Change bind.address from efm.properties
```sh
vim /etc/edb/efm-4.7/efm.properties
bind.address=192.168.5.241:7800         ## standby server ip 
```
- Start Service and configure firewall
```sh
sudo systemctl start edb-efm-4.7.service
sudo firewall-cmd --permanent --add-port={7800,7809}/tcp
sudo firewall-cmd --reload

```
- Check EFM Status 
```sh
/usr/edb/efm-4.7/bin/efm cluster-status efm
```
- Path export on user .bash_profile and reload it 
```sh
export PATH=$PATH:/usr/edb/efm-4.7/bin
```
- allow node 
```sh
/usr/edb/efm-4.7/bin/efm allow-node  efm 192.168.5.241
```

### EFM Configuration on witness Server 
- copy efm configuration file primary to withness node 
```sh
### primary node 
sudo scp efm.nodes efm.properties root@192.168.5.242:/etc/edb/efm-4.7
### withness node 
sudo chown efm:efm /etc/edb/efm-4.7/efm.properties
sudo chown efm:efm /etc/edb/efm-4.7/efm.nodes
```
- Keep everything as it Change bind.address and is.witness from efm.properties
```sh
vim /etc/edb/efm-4.7/efm.properties
bind.address=192.168.5.242:7800         -- standby server ip 
is.witness=true 
```
- Start Service and configure firewall
```sh
sudo systemctl start edb-efm-4.7.service
sudo firewall-cmd --permanent --add-port={7800,7809}/tcp
sudo firewall-cmd --reload

```
- Check EFM Status 
```sh
/usr/edb/efm-4.7/bin/efm cluster-status efm
```
- Path export on user .bash_profile and reload it 
```sh
export PATH=$PATH:/usr/edb/efm-4.7/bin
```
- allow node 
```sh
/usr/edb/efm-4.7/bin/efm allow-node  efm 192.168.5.241
```
- Restart Primary and standby DB and allow note(3 nodes) EFM cluster then check efm 
- Ensure that all efm nodes are allow to efm nodes in EFM cluster
```sh
sudo systemctl restart edb-as-15
/usr/edb/efm-4.7/bin/efm cluster-status efm
/usr/edb/efm-4.7/bin/efm allow-node  efm 192.168.5.240
/usr/edb/efm-4.7/bin/efm allow-node  efm 192.168.5.241
/usr/edb/efm-4.7/bin/efm allow-node  efm 192.168.5.242
```

- Some EFM Command 
```sh
efm promote efm         # Promoted Standby to parimary, Primary will shut down. 
efm promote efm -switchover     # swithover server, If your server is sync mode then ensure both server sync configuration is same
efm set-priority efm ip address     # Change priority Server 
```

- NB 
```t
When You reconfigure EPAS please restart EFM. Make ensure Ports are Open. Ensure All Email address is same. 
Ensure "promote_trigger_file = '/tmp/trigger_file.trg'" this line is entry on both Primary and 
standby server postgresql.conf file. Ensure your network connection speed minimum 100 MB/s.
Log file location /var/log/efm-4.7/
```
