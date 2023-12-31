### Install Barman On RHEL 8 and configure for Replica server with Stream wal replication

#### Prerequisite
- Enable EDB subcription
- Internet Access 
- OS user with sudo privileges
- Python 3.6
- EPAS Binary 
- Key base ssh authentication from barman to DB server
- Barman IP: 192.168.5.242, EPAS IP: 192.168.5.241
- Ensure rsync package installed 


- Login EDB websit (https://www.enterprisedb.com/accounts/profile) 
- Repo Access --> Access EDB Repos 2.0 
- Select Platform --> RHEL 8 --> Select software --> EDB Postgres Advanced Server (with version)
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
ssh-copy-id enterprisedb@192.168.5.241				##  db server user and pass standby
ssh-copy-id enterprisedb@192.168.5.240				##  db server user and pass primary
ssh enterprisedb@192.168.5.241
```
#### On EPAS Server Site 
- Keybase login both server (pirmary & standby)
- DB User create for barman backup
- Barman server IP add in EPAS server pg_hba.conf file 
- Restart EPAS
```sh
### From standby site 
sudo su - enterprisedb
ssh-keygen				## if you already have ssh key no need it 
ssh-copy-id barman@192.168.5.242
ssh  barman@192.168.5.242       ### ssh login with out password

### In Primary Server 
sudo su - enterprisedb 
ssh-keygen				## if you already have ssh key no need it 
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
sudo vim syncstandby.conf

[syncstandby]
description =  "Backup form sync standby Server"
ssh_command = ssh enterprisedb@192.168.5.241    # standby server ip
conninfo = host=192.168.5.241 user=barman port=5444 dbname=edb password=hello  # standby server info
backup_options = concurrent_backup
;backup_method = postgres         ### If you use postgres it't not support reuse_backup = link
backup_method = rsync
streaming_conninfo = host=192.168.5.241 port=5444 user=streaming_barman dbname=edb password=hello  # standby server info
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
barman check syncstandby 
barman show-server syncstandby | grep incoming_wals_directory
## barman show-server command output
## incoming_wals_directory: /var/lib/barman/syncstandby/syncstandby/incoming
```

### EPAS Server Primary 
- Update archive_command for PITR, wal file copy to barman incoming directory (both server )
```sh
# edit postgresql.conf
archive_command = 'test ! -f barman@192.168.5.242:/var/lib/barman/syncstandby/syncstandby/incoming/%f && rsync -a %p barman@192.168.5.242:/var/lib/barman/syncstandby/syncstandby/incoming/%f'
### Archive command for multifile location archive file copy
archive_command = 'rsync -a %p /var/lib/edb/as15/ARCHIVELOG/%f && rsync -a %p barman@192.168.5.242:/var/lib/barman/syncstandby/syncstandby/incoming/%f && rsync -a %p enterprisedb@192.168.5.241:/var/lib/edb/as15/ARCHIVELOG/%f'

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
barman check syncstandby         # Barman configuration check for syncstandby
barman switch-xlog --force --archive syncstandby      # Wal file switch over
barman show-server syncstandby        # show syncstandby Server backup 
barman backup syncstandby         # Backup syncstandby server 
barman list-backup syncstandby    ### output First part is name second part is backup id
barman check-backup syncstandby backup_id         # backup check 
barman show-backup syncstandby latest     # letest backup check 
barman list-files syncstandby backup-id   # file show 

```

#### Scheduling Backups
- Ideally your backups should happen automatically on a schedule

```t
In this step we’ll automate our backups, and we’ll tell Barman to perform maintenance on the backups so files older than the retention policy are deleted. To enable scheduling, run this command as the barman user on the syncstandby (switch to this user if necessary)
```
```sh
sudo su - barman 
crontab -e
30 23 * * * /usr/bin/barman backup syncstandby    # full backup of the syncstandby every night at 11:30 PM
* * * * * /usr/bin/barman cron              # Run every minute and perform maintenance operations on both WAL files and base backup files.
```

#### Restoring or Migrating to a Remote Server

- First stop target database 
```sh
sudo systemctl stop edb-as-15
```
- Let’s locate the details for the latest backup
```sh
barman show-backup syncstandby latest

### output 
Backup 20231029T170617:
  Server Name            : syncstandby
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
Next, run this command to restore the specified backup from the syncstandby to the target server:

```
- Recover PITR
```sh
barman recover --target-time "Begin time"  --remote-ssh-command "ssh enterprisedb@target-db-server-ip"   syncstandby   backup-id   /var/lib/edb/as15/data

### With out PITR
barman recover --remote-ssh-command "ssh enterprisedb@target-db-server-ip"   syncstandby   backup-id   /var/lib/edb/as15/data
```

```t
There are quite a few options, arguments, and variables here, so let’s explain them.

    -- target-time "Begin time": Use the begin time from the show-backup command
    -- remote-ssh-command "ssh postgres@standby-db-server-ip": Use the IP address of the target-db-server-ip
    -- syncstandby: Use the name of the database server from your /etc/barman.conf file
    backup-id: Use the backup ID from the show-backup command, or use latest if that’s the one you want
    /var/lib/edb/as15/data: The path where you want the backup to be restored. This path will become the new data directory for Postgres on the standby server. Here, we have chosen the default data directory for Postgres in CentOS. For real-life use cases, choose the appropriate path

```

- After successfully Restore database start target-db-server service 
```sh
sudo systemctl start edb-as-15
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
