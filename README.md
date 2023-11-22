# How to install Apache Doris on multiple servers.
## Requirements
1. **OS** : CentOS 7.x or Ubuntu 16.04 or higher.
2. **Software** : Java 1.8 and above(jdk and jvm) AND GCC 4.8.2 and above.
3. **Apache Doris** : https://doris.apache.org/download
4. **DB** : MySQL client
5. **Hardware**:<br /> 
   ```Module	  | CPU	      | Memory | Disk                     | Network                 |	Number of Instances```<br />
   ```Frontend	| 16 core + | 64GB + | SSD or RAID card 100GB + | 10,000 Mbp network card	| 1-3*```<br />
   ```Backend	  | 16 core + | 64GB + | SSD or SATA, 100G +      | 10-100 Mbp network card	| 3 * ```<br />

## Installation
### 1. Install java
Install Java and set JAVA_HOME environment variable.
```
$ java -version
$ echo $JAVA_HOME  // should return the jar path
```
### 2. Set open file descriptors in the system
Set the maximum number of open file descriptors in the system. You can change that in the file `/etc/security/limits.conf`.
```
* soft nofile 65536
* hard nofile 65536
```

### 3. Install MySQL
Install mysql installation guide ``` https://www.cyberciti.biz/faq/installing-mysql-server-on-ubuntu-22-04-lts-linux/``` or
download the installation-free MySQL client ```https://doris-build-hk.oss-cn-hongkong.aliyuncs.com/mysql-client/mysql-5.7.22-linux-glibc2.12-x86_64.tar.gz```

### 4. Install Apache Doris
To run Apache Doris you will have to install and run
- Frontend(fe)
- Backend(be)
  
This will come in the tar file.

#### 4.1 Download and Unzip
Download the latest version of Doris `https://doris.apache.org/download` and unzip the tar file ```tar zxf apache-doris-x.x.x.tar.gz```

#### 4.2 Frontend - Configure and run
- cd into `fe` folder `$ cd apache-doris-x.x.x/fe`
- Modify the FE configuration file `vi conf/fe.conf`
- Mainly modify two parameters: `priority_networks` and `meta_dir`
- If you need more optimized configuration, please refer to FE parameter configuration: `https://doris.apache.org/docs/1.2/admin-manual/config/fe-config` for instructions on how to adjust them.

##### 4.2.1 Modify Conf file for FE
```
priority_networks=172.23.16.0/24
meta_dir=/path/your/doris-meta
```
###### **Note 1**:- `priority_networks` parameter we have to configure during installation, especially when a machine has multiple IP addresses, we have to specify a unique IP address for FE.
###### **Note 2**:- Here you can leave `meta_dir` unconfigured, the default is doris-meta in your Doris FE installation directory. To configure the metadata directory separately, you need to create the directory you specify in advance

##### 4.2.2 Run Fe (Frontend)
Inside the `fe` folder run the command
```
./bin/start_fe.sh --daemon
```

View FE operational status
```
curl http://127.0.0.1:8030/api/bootstrap
```

If the return result has the word `"msg": "success"`, then the startup was successful.
You can also check this through the web UI provided by Doris FE by entering the address in your browser
```
http:// fe_ip:8030
```

###### Note1:- Here we use the Doris built-in default user, root, to log in with an empty password.
###### Note2:- This is an administrative interface for Doris, and only users with administrative privileges can log in.

##### 4.2.3 Connect FE
We will connect to Doris FE via MySQL client
```
mysql -uroot -P9030 -h127.0.0.1
```
###### Note1:-  The root user used here is the default user built into doris, and is also the super administrator user, see Rights Management
###### Note2:-  -P: Here is our query port to connect to Doris, the default port is 9030, which corresponds to query_port in fe.conf
###### Note3:-  -h: Here is the IP address of the FE we are connecting to, if your client and FE are installed on the same node you can use 127.0.0.1, this is also provided by Doris if you forget the root password, you can connect directly to the login without the password in this way and reset the root password.

Execute the following command to view the FE running status
```
show frontends\G;
```

You can then see a result similar to the following.
```
mysql> show frontends\G;
*************************** 1. row ***************************
             Name: 172.21.32.5_9010_1660549353220
               IP: 172.21.32.5
      EditLogPort: 9010
         HttpPort: 8030
        QueryPort: 9030
          RpcPort: 9020
             Role: FOLLOWER
         IsMaster: true
        ClusterId: 1685821635
             Join: true
            Alive: true
ReplayedJournalId: 49292
    LastHeartbeat: 2022-08-17 13:00:45
         IsHelper: true
           ErrMsg:
          Version: 1.1.2-rc03-ca55ac2
 CurrentConnected: Yes
1 row in set (0.03 sec)
```
1. If the IsMaster, Join and Alive columns are true, the node is normal.

#####  4.2.4 SSl for FE MySql
Communicate with the server over an encrypted connection
Doris supports SSL-based encrypted connections. It currently supports TLS1.2 and TLS1.3 protocols. Doris' SSL mode can be enabled through the following configuration: Modify the FE configuration file `conf/fe.conf` and add `enable_ssl = true`.
Refer: `https://doris.apache.org/docs/1.2/get-starting/#start-fe`

##### 4.2.5 Scale FE Frontend
To scale FE refer: `https://doris.apache.org/docs/dev/admin-manual/cluster-management/elastic-expansion/`

##### 4.2.6 Stop FE Frontend
```
$ ./bin/stop_fe.sh
```


#### 4.3 Backend - Configure and run
- Go to the `apache-doris-x.x.x/be` directory
- Modify the BE configuration file `conf/be.conf`
- Mainly modify two parameters: `priority_networks` and `storage_root`
- if you need more optimized configuration, please refer to BE parameter configuration: `https://doris.apache.org/docs/1.2/admin-manual/config/be-config` instructions to make adjustments.

##### 4.3.1 Modify Conf file for BE
```
priority_networks=172.23.16.0/24
storage_root_path=/path/your/data_dir
```
###### Note1:- For 'storage_root_path', the default directory is in the storage directory of the BE installation directory. The storage directory for BE configuration must be created first

##### 4.3.2 Start BE
Execute the following command in the BE installation directory to complete the BE startup.
```
$ ./bin/start_be.sh --daemon
```

##### 4.3.3 Add BE node to Apache Doris cluster
Connect to FE via MySQL client and execute the following SQL to add the BE to the cluster/FE
```
ALTER SYSTEM ADD BACKEND "be_host_ip:heartbeat_service_port";
```
1. be_host_ip: Here is the IP address of your BE, match with `priority_networks` in `be.conf`.
2. heartbeat_service_port: This is the heartbeat upload port of your BE, match with `heartbeat_service_port` in `be.conf`, default is `9050`.

Check if the node is operational
```
SHOW BACKENDS\G；
```
Example:

```
mysql> SHOW BACKENDS\G;
*************************** 1. row ***************************
            BackendId: 10003
              Cluster: default_cluster
                   IP: 172.21.32.5
        HeartbeatPort: 9050
               BePort: 9060
             HttpPort: 8040
             BrpcPort: 8060
        LastStartTime: 2022-08-16 15:31:37
        LastHeartbeat: 2022-08-17 13:33:17
                Alive: true
 SystemDecommissioned: false
ClusterDecommissioned: false
            TabletNum: 170
     DataUsedCapacity: 985.787 KB
        AvailCapacity: 782.729 GB
        TotalCapacity: 984.180 GB
              UsedPct: 20.47 %
       MaxDiskUsedPct: 20.47 %
                  Tag: {"location" : "default"}
               ErrMsg:
              Version: 1.1.2-rc03-ca55ac2
               Status: {"lastSuccessReportTabletsTime":"2022-08-17 13:33:05","lastStreamLoadTime":-1,"isQueryDisabled":false,"isLoadDisabled":false}
1 row in set (0.01 sec)
```
Alive : true means the node is running normally

##### 4.3.4 Stop BE
The stopping of Doris BE can be done with the following command
```
$ ./bin/stop_be.sh
```

## Reference
1. https://doris.apache.org/docs/1.2/install/standard-deployment/
2. https://doris.apache.org/docs/1.2/get-starting/
