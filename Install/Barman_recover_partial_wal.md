### Install Barman On RHEL 8 and configure wal replication and recover from partial wal file

#### Prerequisite
- Enable EDB subcription
- Internet Access 
- OS user with sudo privileges
- Python 3.6
- EPAS Binary 
- Key base ssh authentication from both side barman to DB server, Recovery server 
- Barman IP: 192.168.5.242, EPAS IP: 192.168.5.240, Recovery Server IP: 192.168.5.239
- Install Barman binary cli on Recovery Server
- Ensure rsync package installed all server
- Install EPAS Binary on Recovery Server 


- Login EDB websit (https://www.enterprisedb.com/accounts/profile) 
- Repo Access --> Access EDB Repos 2.0 
- Select Platform --> RHEL 8 --> Select software --> EDB Postgres Advanced Server (with version)
#### On Barman Server Site 
```sh
sudo curl -1sLf 'https://downloads.enterprisedb.com/zp579PrIC9a7kY4rQtxX63HAaXHtzeCA/enterprise/setup.rpm.sh' | sudo -E bash
sudo dnf -y install edb-as15-server
sudo dnf -y module install python36
sudo dnf -y install barman barman-cli
id barman
barman --version
sudo passwd barman
sudo su - barman
ssh-keygen
ssh-copy-id enterprisedb@192.168.5.240				##  db server user and pass primary
ssh-copy-id enterprisedb@192.168.5.239				##  db server user and pass Recovery Server
ssh enterprisedb@192.168.5.240
ssh enterprisedb@192.168.5.242
```
#### On EPAS Server Site 
- Keybase login  
- DB User create for barman backup
- Barman server IP add in EPAS server pg_hba.conf file 
- Restart EPAS
```sh
sudo su - enterprisedb 
ssh-keygen				
ssh-copy-id barman@192.168.5.242
ssh  barman@192.168.5.242       ### ssh login with out password
psql edb 
create user barman superuser  password 'hello';
CREATE ROLE streaming_barman WITH REPLICATION PASSWORD 'hello' LOGIN;
\q
```
- Update pg_hba.conf replication server add barman replication server ip then reload 
```sh
host  replication     streaming_barman 192.168.5.242/32         scram-sha-256
host    all             all            192.168.5.242/32           scram-sha-256
```

#### Barman server 
- Copy barman.con file for backup 
- Update barman.conf file with below information
- Create conf file file which contain backup information
```sh
sudo cp /etc/barman.conf	/etc/barman.conf.orginal
sudo vim /etc/barman.conf

barman_home = /var/lib/barman/     # Ensure Directory is exsis
compression = gzip
immediate_checkpoint = true
basebackup_retry_times = 3
reuse_backup = link         ## Add this entry, for rsync 


cd /etc/barman.d
sudo vim partialwal.conf

[partialwal]
description =  "Backup from EPAS Server"
ssh_command = ssh enterprisedb@192.168.5.240   # EPAS server ip
conninfo = host=192.168.5.240 user=barman port=5444 dbname=edb password=hello  # EPAS server info
backup_options = concurrent_backup
;backup_method = postgres         ### If you use postgres it't not support reuse_backup = link
backup_method = rsync
streaming_conninfo = host=192.168.5.240 port=5444 user=streaming_barman dbname=edb password=hello  # EPAS server info
streaming_archiver = on
archiver = on
slot_name = barman
create_slot = auto
path_prefix = "/usr/edb/as15/bin"
streaming_archiver_name = barman_receive_wal
```
- First, we need to locate the value of the incoming backup directory from the barman-backup-server. On the Barman server, switch to the user barman
```sh
sudo su - barman
barman check partialwal 
barman show-server partialwal | grep incoming_wals_directory
## barman show-server command output
## incoming_wals_directory: /var/lib/barman/partialwal/partialwal/incoming
```

### EPAS Server 
- Update archive_command for PITR, wal file copy to barman incoming directory
```sh
# edit postgresql.conf
archive_command = 'test ! -f barman@192.168.5.242:/var/lib/barman/partialwal/partialwal/incoming/%f && rsync -a %p barman@192.168.5.242:/var/lib/barman/partialwal/partialwal/incoming/%f'
### Archive command for multifile location archive file copy
archive_command = 'rsync -a %p /var/lib/edb/as15/ARCHIVELOG/%f && rsync -a %p barman@192.168.5.242:/var/lib/barman/partialwal/partialwal/incoming/%f && rsync -a %p enterprisedb@192.168.5.241:/var/lib/edb/as15/ARCHIVELOG/%f'

### NB: Ensure all server can ssh access from eatch other without password 

sudo systemctl restart edb-as-15
sudo su - enterprisedb 
psql edb 
select pg_switch_wal();	        ## Switch wal file and check it 
```
### Testing Barman 
```sh
sudo su - barman            # switch barman user 
barman list-server          # server list 
barman check partialwal         # Barman configuration check for partialwal
barman switch-xlog --force --archive partialwal      # Wal file switch over
barman show-server partialwal        # show partialwal Server backup 
barman backup partialwal         # Backup partialwal server 
barman list-backup partialwal    ### output First part is name second part is backup id
barman check-backup partialwal backup_id         # backup check 
barman show-backup partialwal latest     # letest backup check 
barman list-files partialwal backup-id   # file show 

```

#### Scheduling Backups
- Ideally your backups should happen automatically on a schedule

```t
In this step we’ll automate our backups, and we’ll tell Barman to perform maintenance on the backups so files older than the retention policy are deleted. To enable scheduling, run this command as the barman user on the partialwal (switch to this user if necessary)
```
```sh
sudo su - barman 
crontab -e
30 23 * * * /usr/bin/barman backup partialwal    # full backup of the partialwal every night at 11:30 PM
* * * * * /usr/bin/barman cron              # Run every minute and perform maintenance operations on both WAL files and base backup files.
```

#### Restoring or Migrating to a Remote Server

#### On Recovery Server
- Install Barman cli
```sh
dnf -y install barman-cli
```
- stop target database if running 
```sh
sudo systemctl stop edb-as-15
```

- TO see pg_replication_slots on standby server 
```sql
select * from pg_replication_slots;
select * from pg_stat_replication ;
```
- To see streaming working from barman
```sql
psql -U streaming_barman -h 192.168.5.240 -c 'IDENTIFY_SYSTEM' replication=1
```
- password less connection from barman to DB server(DB user Passwod)
```sh
su - barman
vim ~/.pgpass
hostname:port:database:username:password
```
- Barman command 
```sh
 barman replication-status standby
 barman receive-wal standby
 barman switch-wal --force --archive standby
 barman list-backup standby
 barman backup standby
```


#### Barman Server 
- Let’s locate the details for the latest backup
```sh
barman show-backup partialwal latest

### output 
Backup 20231029T170617:
  Server Name            : partialwal
  System Id              : 7294227102000760227
  Status                 : DONE
  PostgreSQL Version     : 150004
  PGDATA directory       : /var/lib/edb/as15/data

  Base backup information:
    Disk usage           : 96.7 MiB (96.7 MiB with WALs)
    Incremental size     : 29.7 MiB (-69.30%)
    Timeline             : 1
    Begin WAL            : 000000010000000000000011
    End WAL              : 000000010000000000000011
    WAL number           : 1
    WAL compression ratio: 99.90%
    Begin time           : 2023-10-29 17:06:17.225323+06:00
    End time             : 2023-10-29 17:06:20.062575+06:00
    Copy time            : 1 second
    Estimated throughput : 19.7 MiB/s

```
```t
From the output, note down the backup ID printed on the first line. Backup 20231029T170617
Also check when the backup was made, from the Begin time filed 
Next, run this command to restore the specified backup from the partialwal to the target server:

```
- Recover With partial wal file 
```sh
barman recover  --get-wal --remote-ssh-command "ssh enterprisedb@192.168.5.239"   partialwal  backup-id   /var/lib/edb/as15/data
```

```t
There are quite a few options, arguments, and variables here, so let’s explain them.

    -- get-wal  Its make recovery comman for recover wal file
    -- remote-ssh-command "ssh postgres@standby-db-server-ip": Use the IP address of the target-db-server-ip
    -- partialwal: Use the name of the database server from your /etc/barman.conf file
    backup-id: Use the backup ID from the show-backup command, or use latest if that’s the one you want
    /var/lib/edb/as15/data: The path where you want the backup to be restored. This path will become the new data directory for Postgres on the standby server. Here, we have chosen the default data directory for Postgres in CentOS. For real-life use cases, choose the appropriate path

```
#### In Recovery Server 
- After successfully Restore database start target-db-server service 
- Go to PG_DATA Directory and modifi postgresql.auto.conf file
- In this file you find restore_command and modifi it 
```sh
su - enterprisedb
cd $PG_DATA
vim postgresql.auto.conf
restore_command = 'barman-wal-restore -P -U barman 192.168.5.242 partialwal %f %p'
### 192.168.5.242 is Barman server ip, partialwal is barman backup name, barman-wal-restore is command 
### You must install barman cli in this server 
### Please change postgresql.conf file accoring to your requrement 
```
- Start EPAS Service 
```sh
sudo systemctl start edb-as-15.service
```
