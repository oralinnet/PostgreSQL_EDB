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

- In my case I used Three Servers 
```t
Master Node (node1) - 192.168.5.240
Standby Node (node2)- 192.168.5.240
Witness Node (node3) - 192.168.5.240
VIP - 10.10.20.5
```
- Install Java on all servers 
```sh
dnf install java -y
java -version
```
- Update pg_hba.conf file on both primary and standby server with the information of the replication and witness servers and reload it
- withness server no need EPAS only EFM install there 