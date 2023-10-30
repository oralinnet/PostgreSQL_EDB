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
- Adjust firewall to allow EFM agents to communicate on ports 7800 and 7810

- In my case I used Three Servers 
```t
Master Node (node1) - 192.168.5.240     -- Primary DB
Standby Node (node2)- 192.168.5.240     -- Standby DB
Witness Node (node3) - 192.168.5.240
VIP - 192.168.5.243
```
- Install Java on all servers 
```sh
dnf install java -y
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
dnf -y install edb-efm47
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
sudo firewall-cmd --permanent --add-port={7800,7810}/tcp
sudo firewall-cm --reload

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
- Stop Primary DB Server and restar EFM 
```sh
sudo systemctl restart edb-as-15
sudo systemctl restart edb-efm-4.7.service
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
bind.address=192.168.5.241:7800         -- standby server ip 
```
- Start Service and configure firewall
```sh
sudo systemctl start edb-efm-4.7.service
sudo firewall-cmd --permanent --add-port=7800/tcp
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
- Stop Primary DB Server and restar EFM 
```sh
sudo systemctl restart edb-as-15
sudo systemctl restart edb-efm-4.7.service
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
sudo firewall-cmd --permanent --add-port=7800/tcp
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
- Stop Primary DB Server and restar EFM 
```sh
sudo systemctl restart edb-as-15
sudo systemctl restart edb-efm-4.7.service
```

- Some EFM Command 
```sh
efm promote efm         # switchover
auto.resume.period      # 
efm set-priority efm ip address     # Change priority Server 
```

