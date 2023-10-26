### Install Barman On RHEL 8

#### Prerequisite
- Enable EDB subcription
- Internet Access 
- OS user with sudo privileges
- Python 3.6
- EPAS Binary 
- Key base ssh auth from barman to DB server
- Barman IP: 192.168.5.242, EPAS IP: 192.168.5.241
- Ensure rsync package installed 


- Login EDB websit (https://www.enterprisedb.com/accounts/profile) 
- Repo Access --> Access EDB Repos 2.0 
- Select Platform --> RHEL 8 --> Select software --> EDB Postgres Advanced Server (with version)
```sh
sudo curl -1sLf 'https://downloads.enterprisedb.com/zp579PrIC9a7kY4rQtxX63HAaXHtzeCA/enterprise/setup.rpm.sh' | sudo -E bash
sudo dnf -y install edb-as15-server
sudo yum module install python36
sudo yum install barman barman-cli
id barman
barman --version
sudo passwd barman
sudo su - barman
ssh-keygen
ssh-copy-id enterprisedb@192.168.5.241				##  db server user and pass
ssh enterprisedb@192.168.5.241
```
- On EPAS Server Site 
```sh
sudo su - enterprisedb
ssh-keygen				## if you already have ssh key
ssh-copy-id barman@192.168.5.242
check ssh login 
psql edb
create user barman superuser  password 'hello';

# Update pg_hba.conf and add barman server ip then reload 
host    all             all            192.168.5.242/32           scram-sha-256

sudo systemctl restart edb-as-15
```

#### Barman server 
- Copy barman.con file for backup 
- Update barman.conf file with below information
```sh
sudo cp /etc/barman.conf	/etc/barman.conf.orginal
sudo vim /etc/barman.conf
barman_home = /var/lib/barman/edb01     # Ensure Directory is exsis
compression = gzip
immediate_checkpoint = true
basebackup_retry_times = 3
reuse_backup = link

sudo cd /etc/barman.d
sudo vim edb01.conf

[edb01]
description =  "EDB01 Server backup"
ssh_command = ssh enterprisedb@192.168.5.241        # EPAS Server ssh 
conninfo = host=192.168.5.241 user=barman port=5444 dbname=edb password=hello   # EPAS Server info
backup_options = concurrent_backup
backup_method = rsync
archiver = on
```

### EPAS Server 
```sh
# edit postgresql.conf
archive_command = 'test ! -f barman@192.168.5.242:/var/lib/barman/edb01/edb01/incoming/%f && rsync -a %p barman@192.168.5.242:/var/lib/barman/edb01/edb01/incoming/%f'

sudo systemctl restart edb-as-15
sudo su - enterprisedb 
psql edb 
select pg_switch_wal();	        ## Switch wal file and check it 
```

### Barman Server 
```sh
sudo su - barman
barman check edb01         -- Barman conf check for edb01
barman switch-xlog --force --archive edb01
barman show-server edb01
barman backup edb01
barman check-backup
barman check-backup edb01

```

```t

```