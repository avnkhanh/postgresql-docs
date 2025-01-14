# Deploying PostgreSQL for high availability with Patroni on Debian or Ubuntu

This guide provides instructions on how to set up a highly available PostgreSQL cluster with Patroni on Debian or Ubuntu. 


## Preconditions

1. This is an example deployment where etcd runs on the same host machines as the Patroni and PostgreSQL and there is a single dedicated HAProxy host. Alternatively etcd can run on different set of nodes. 

    If etcd is deployed on the same host machine as Patroni and PostgreSQL, separate disk system for etcd and PostgreSQL is recommended due to performance reasons.

2. For this setup, we will use the nodes running on Ubuntu 22.04 as the base operating system:

| Node         | Components                           | IP Address       |
|--------------|--------------------------------------|------------------|
| 111pgsql     | PostgreSQL + Patroni + etcd          | 192.168.55.111   |
| 112pgsql     | PostgreSQL + Patroni + etcd          | 192.168.55.112   |
| 113pgsql     | PostgreSQL + Patroni + etcd          | 192.168.55.113   |
| 110pgsqlha   | HAProxy load balancer                | 192.168.55.110   |

!!! note

    We recommend not to expose the hosts/nodes where Patroni / etcd / PostgreSQL are running to public networks due to security risks.  Use Firewalls, Virtual networks, subnets or the like to protect the database hosts from any kind of attack. 

## Initial setup 

Configure every node.

### Set up hostnames in the `/etc/hosts` file

It's not necessary to have name resolution, but it makes the whole setup more readable and less error prone. Here, instead of configuring a DNS, we use a local name resolution by updating the file `/etc/hosts`. By resolving their hostnames to their IP addresses, we make the nodes aware of each other's names and allow their seamless communication. 

=== "111pgsql"   

    1. Set up the hostname for the node

        ```{.bash data-prompt="$"}
        sudo hostnamectl set-hostname 111pgsql
        ```

    2. Modify the `/etc/hosts` file to include the hostnames and IP addresses of the remaining nodes. Add the following at the end of the `/etc/hosts` file on all nodes:   

        ```text hl_lines="3 4"
        # Cluster IP and names 
        192.168.55.111 111pgsql 
        192.168.55.112 112pgsql 
        192.168.55.113 113pgsql
        ```    

=== "112pgsql"    

    1. Set up the hostname for the node

        ```{.bash data-prompt="$"}
        sudo hostnamectl set-hostname 112pgsql
        ```

    2. Modify the `/etc/hosts` file to include the hostnames and IP addresses of the remaining nodes. Add the following at the end of the `/etc/hosts` file on all nodes:   

        ```text hl_lines="2 4"
        # Cluster IP and names 
        192.168.55.111 111pgsql 
        192.168.55.112 112pgsql 
        192.168.55.113 113pgsql
        ```    

=== "113pgsql"    

    1. Set up the hostname for the node

        ```{.bash data-prompt="$"}
        sudo hostnamectl set-hostname 113pgsql
        ```

    2. Modify the `/etc/hosts` file to include the hostnames and IP addresses of the remaining nodes. Add the following at the end of the `/etc/hosts` file on all nodes:   

        ```text hl_lines="2 3"
        # Cluster IP and names 
        192.168.55.111 111pgsql 
        192.168.55.112 112pgsql 
        192.168.55.113 113pgsql
        ```    

=== "110pgsqlha"  

    1. Set up the hostname for the node

        ```{.bash data-prompt="$"}
        sudo hostnamectl set-hostname 110pgsqlha
        ```

    2. Modify the `/etc/hosts` file. The HAProxy instance should have the name resolution for all the three nodes in its `/etc/hosts` file. Add the following lines at the end of the file:    

        ```text hl_lines="3 4 5"
        # Cluster IP and names
        192.168.55.110 110pgsqlha
        192.168.55.111 111pgsql
        192.168.55.112 112pgsql
        192.168.55.113 113pgsql
        ```

### Install the software

Run the following commands on `111pgsql`, `112pgsql` and `113pgsql`:

1. Install Percona Distribution for PostgreSQL
    
    * Disable the upstream `postgresql-16` package.

    * Install the `percona-release` repository management tool
    
    ```{.bash data-prompt="$"}
    sudo apt install -y wget curl lsb-release nfs-common net-tools
	curl -O https://repo.percona.com/apt/percona-release_latest.generic_all.deb
	chmod 644 /root/percona-release_latest.generic_all.deb
	sudo apt install -y gnupg2 lsb-release ./percona-release_latest.generic_all.deb
	sudo apt update
    ```

    * Enable the repository

        ```{.bash data-prompt="$"}
        sudo percona-release setup ppg16
        ```   

    * Install Percona Distribution for PostgreSQL package

        ```{.bash data-prompt="$"}
        sudo apt install percona-postgresql-16
        ```

2. Install some Python and auxiliary packages to help with Patroni and etcd
    
    ```{.bash data-prompt="$"}
    sudo apt install python3-pip python3-dev binutils
    ```

3. Install etcd, Patroni, pgBackRest packages:


    ```{.bash data-prompt="$"}
    sudo apt install percona-patroni \
    etcd etcd-server etcd-client \
    percona-pgbackrest
    ```

4. Stop and disable all installed services:
    
    ```{.bash data-prompt="$"}
    sudo systemctl stop {etcd,patroni,postgresql}
    systemctl disable {etcd,patroni,postgresql}
    ```

5. Even though Patroni can use an existing Postgres installation, remove the data directory to force it to initialize a new Postgres cluster instance.

   ```{.bash data-prompt="$"}
   sudo systemctl stop postgresql
   sudo rm -rf /var/lib/postgresql/16/main
   sudo rm -rf /etc/postgresql/16/main
   ```

## Configure etcd distributed store  

In our implementation we use etcd distributed configuration store. [Refresh your knowledge about etcd](high-availability.md#etcd).

!!! note
    
    If you [installed the software from tarballs](../tarball.md), you must first [enable etcd](../enable-extensions.md#etcd) before configuring it.

To get started with `etcd` cluster, you need to bootstrap it. This means setting up the initial configuration and starting the etcd nodes so they can form a cluster. There are the following bootstrapping mechanisms:  

* Static in the case when the IP addresses of the cluster nodes are known
* Discovery service - for cases when the IP addresses of the cluster are not known ahead of time.
    
Since we know the IP addresses of the nodes, we will use the static method. For using the discovery service, please refer to the [etcd documentation :octicons-external-link-16:](https://etcd.io/docs/v3.5/op-guide/clustering/#etcd-discovery){:target="_blank"}.

We will configure and start all etcd nodes in parallel. This can be done either by modifying each node's configuration or using the command line options. Use the method that you prefer more.

### Method 1. Modify the configuration file

1. Create the etcd configuration file on every node. You can edit the sample configuration file `/etc/etcd/etcd.conf.yaml` or create your own one. Replace the node names and IP addresses with the actual names and IP addresses of your nodes.

    === "111pgsql"

         ```yaml title="/etc/etcd/etcd.conf.yaml"
         name: '111pgsql'
         initial-cluster-token: PostgreSQL_HA_Cluster_1
         initial-cluster-state: new
         initial-cluster: 111pgsql=http://192.168.55.111:2380,112pgsql=http://192.168.55.112:2380,113pgsql=http://192.168.55.113:2380
         data-dir: /var/lib/etcd
         initial-advertise-peer-urls: http://192.168.55.111:2380 
         listen-peer-urls: http://192.168.55.111:2380
         advertise-client-urls: http://192.168.55.111:2379
         listen-client-urls: http://192.168.55.111:2379
         ```

    === "112pgsql"

         ```yaml title="/etc/etcd/etcd.conf.yaml"
         name: '112pgsql'
         initial-cluster-token: PostgreSQL_HA_Cluster_1
         initial-cluster-state: new
         initial-cluster: 111pgsql=http://192.168.55.111:2380,112pgsql=http://192.168.55.112:2380,     113pgsql=http://192.168.55.113:2380
         data-dir: /var/lib/etcd
         initial-advertise-peer-urls: http://192.168.55.112:2380 
         listen-peer-urls: http://192.168.55.112:2380
         advertise-client-urls: http://192.168.55.112:2379
         listen-client-urls: http://192.168.55.112:2379
         ```

    === "113pgsql"

         ```yaml title="/etc/etcd/etcd.conf.yaml"
         name: '113pgsql'
         initial-cluster-token: PostgreSQL_HA_Cluster_1
         initial-cluster-state: new
         initial-cluster: 111pgsql=http://192.168.55.111:2380,112pgsql=http://192.168.55.112:2380,     113pgsql=http://192.168.55.113:2380
         data-dir: /var/lib/etcd
         initial-advertise-peer-urls: http://192.168.55.113:2380 
         listen-peer-urls: http://192.168.55.113:2380
         advertise-client-urls: http://192.168.55.113:2379
         listen-client-urls: http://192.168.55.113:2379
         ```

2. Enable and start the `etcd` service on all nodes:

    ```{.bash data-prompt="$"}
    sudo systemctl enable --now etcd
    sudo systemctl start etcd
    sudo systemctl status etcd
    ```

    During the node start, etcd searches for other cluster nodes defined in the configuration. If the other nodes are not yet running, the start may fail by a quorum timeout. This is expected behavior. Try starting all nodes again at the same time for the etcd cluster to be created.

    * Thiết lập biến môi trường và kiểm tra cụm etcd
    ```
	export ETCDCTL_API=3
	HOST_1=192.168.55.111
	HOST_2=192.168.55.112
	HOST_3=192.168.55.113
	ENDPOINTS=$HOST_1:2379,$HOST_2:2379,$HOST_3:2379
    ```

    * Kiểm tra thành viên cụm:
    ```
	sudo etcdctl --endpoints=$ENDPOINTS -w table member list
    ```


    * Kiểm tra node leader
    ```
	sudo etcdctl --endpoints=$ENDPOINTS -w table endpoint status
    ```

### Method 2. Start etcd nodes with command line options

1. On each etcd node, set the environment variables for the cluster members, the cluster token and state:

    ```
    TOKEN=PostgreSQL_HA_Cluster_1
    CLUSTER_STATE=new
    NAME_1=111pgsql
    NAME_2=112pgsql
    NAME_3=113pgsql
    HOST_1=192.168.55.111
    HOST_2=192.168.55.112
    HOST_3=192.168.55.113
    CLUSTER=${NAME_1}=http://${HOST_1}:2380,${NAME_2}=http://${HOST_2}:2380,${NAME_3}=http://${HOST_3}:2380
    ```

2. Start each etcd node in parallel using the following command:

    === "111pgsql"

        ```{.bash data-prompt="$"}
        THIS_NAME=${NAME_1}
        THIS_IP=${HOST_1}
        etcd --data-dir=data.etcd --name ${THIS_NAME} \
        	--initial-advertise-peer-urls http://${THIS_IP}:2380 --listen-peer-urls http://${THIS_IP}:2380 \
        	--advertise-client-urls http://${THIS_IP}:2379 --listen-client-urls http://${THIS_IP}:2379 \
        	--initial-cluster ${CLUSTER} \
        	--initial-cluster-state ${CLUSTER_STATE} --initial-cluster-token ${TOKEN}
        ```

    === "112pgsql"

        ```{.bash data-prompt="$"}
        THIS_NAME=${NAME_2}
        THIS_IP=${HOST_2}
        etcd --data-dir=data.etcd --name ${THIS_NAME} \
        	--initial-advertise-peer-urls http://${THIS_IP}:2380 --listen-peer-urls http://${THIS_IP}:2380 \
        	--advertise-client-urls http://${THIS_IP}:2379 --listen-client-urls http://${THIS_IP}:2379 \
        	--initial-cluster ${CLUSTER} \
        	--initial-cluster-state ${CLUSTER_STATE} --initial-cluster-token ${TOKEN}
        ```

    === "113pgsql"

        ```{.bash data-prompt="$"}
        THIS_NAME=${NAME_3}
        THIS_IP=${HOST_3}
        etcd --data-dir=data.etcd --name ${THIS_NAME} \
        	--initial-advertise-peer-urls http://${THIS_IP}:2380 --listen-peer-urls http://${THIS_IP}:2380 \
        	--advertise-client-urls http://${THIS_IP}:2379 --listen-client-urls http://${THIS_IP}:2379 \
        	--initial-cluster ${CLUSTER} \
        	--initial-cluster-state ${CLUSTER_STATE} --initial-cluster-token ${TOKEN}
        ```

    * Thiết lập biến môi trường và kiểm tra cụm etcd
    ```
	export ETCDCTL_API=3
	HOST_1=192.168.55.111
	HOST_2=192.168.55.112
	HOST_3=192.168.55.113
	ENDPOINTS=$HOST_1:2379,$HOST_2:2379,$HOST_3:2379
    ```

    * Kiểm tra thành viên cụm:
    ```
	sudo etcdctl --endpoints=$ENDPOINTS -w table member list
    ```


    * Kiểm tra node leader
    ```
	sudo etcdctl --endpoints=$ENDPOINTS -w table endpoint status
    ```

## Configure Patroni

Run the following commands on all nodes. You can do this in parallel:

1. Export and create environment variables to simplify the config file creation:

    * Xuất và tạo các biến môi trường để đơn giản hóa việc tạo tệp cấu hình:
	```{.bash data-prompt="$"}	
	export NODE_NAME=`hostname -f`
	export NODE_IP=`hostname -i | awk '{print $1}'`
	DATA_DIR="/var/lib/postgresql/16/main"
	PG_BIN_DIR="/usr/lib/postgresql/16/bin"
	NAMESPACE="pnc_namespace"
	SCOPE="postgres_scope"
	```

2. Use the following command to create the `/etc/patroni/patroni.yml` configuration file and add the following configuration for `111pgsql`:

    ```bash
    echo "
    namespace: ${NAMESPACE}
    scope: ${SCOPE}
    name: ${NODE_NAME}

    restapi:
        listen: 0.0.0.0:8008
        connect_address: ${NODE_IP}:8008

    etcd3:
        host: ${NODE_IP}:2379

    bootstrap:
      # this section will be written into Etcd:/<namespace>/<scope>/config after initializing new cluster
      dcs:
          ttl: 30
          loop_wait: 10
          retry_timeout: 10
          maximum_lag_on_failover: 1048576

          postgresql:
              use_pg_rewind: true
              use_slots: true
              parameters:
                  wal_level: replica
                  hot_standby: "on"
                  wal_keep_segments: 10
                  max_wal_senders: 5
                  max_replication_slots: 10
                  wal_log_hints: "on"
                  logging_collector: 'on'
                  max_wal_size: '10GB'
                  archive_mode: "on"
                  archive_timeout: 600s
                  archive_command: pgbackrest --stanza=cluster_1 archive-push /var/lib/postgresql/16/main/pg_wal/%f
              recovery_conf:
                  restore_command: pgbackrest --config=/etc/pgbackrest/pgbackrest.conf --stanza=cluster_1 archive-get %f %p

      # some desired options for 'initdb'
      initdb: # Note: It needs to be a list (some options need values, others are switches)
          - encoding: UTF8
          - data-checksums

      pg_hba: # Add following lines to pg_hba.conf after running 'initdb'
          - host replication replicator 127.0.0.1/32 trust
          - host replication replicator 0.0.0.0/0 md5
          - host all all 0.0.0.0/0 md5
          - host all all ::0/0 md5

      # Some additional users which needs to be created after initializing new cluster
      users:
          admin:
              password: s2Min123
              options:
                  - createrole
                  - createdb
          percona:
              password: perc0n2123
              options:
                  - createrole
                  - createdb 
    
    postgresql:
        cluster_name: cluster_1
        listen: 0.0.0.0:5432
        connect_address: ${NODE_IP}:5432
        data_dir: ${DATA_DIR}
        bin_dir: ${PG_BIN_DIR}
        pgpass: /tmp/pgpass0
        authentication:
            replication:
                username: replicator
                password: replP2sswd
            superuser:
                username: postgres
                password: p0stGres123
        parameters:
            unix_socket_directories: "/var/run/postgresql/"
        create_replica_methods:
            - basebackup
        basebackup:
            checkpoint: 'fast'

    tags:
        nofailover: false
        noloadbalance: false
        clonefrom: false
        nosync: false
    " | sudo tee -a /etc/patroni/patroni.yml
    ```

    ??? admonition "Patroni configuration file"

        Let’s take a moment to understand the contents of the `patroni.yml` file. 

        The first section provides the details of the node and its connection ports. After that, we have the `etcd` service and its port details.

        Following these, there is a `bootstrap` section that contains the PostgreSQL configurations and the steps to run once the database is initialized. The `pg_hba.conf` entries specify all the other nodes that can connect to this node and their authentication mechanism. 


3. Check that the systemd unit file `percona-patroni.service` is created in `/etc/systemd/system`. If it is created, skip this step. 

   If it's **not created**, create it manually and specify the following contents within:

    ```ini title="/etc/systemd/system/percona-patroni.service"
    [Unit]
     Description=Runners to orchestrate a high-availability PostgreSQL
     After=syslog.target network.target 

     [Service]
     Type=simple 

     User=postgres
     Group=postgres 

     # Start the patroni process
     ExecStart=/bin/patroni /etc/patroni/patroni.yml 

     # Send HUP to reload from patroni.yml
     ExecReload=/bin/kill -s HUP $MAINPID 

     # only kill the patroni process, not its children, so it will gracefully stop postgres
     KillMode=process 

     # Give a reasonable amount of time for the server to start up/shut down
     TimeoutSec=30 

     # Do not restart the service if it crashes, we want to manually inspect database on failure
     Restart=no 

     [Install]
     WantedBy=multi-user.target
    ```

4. Make systemd aware of the new service:

    ```{.bash data-prompt="$"}
    sudo systemctl daemon-reload
    ```

5. Repeat steps 1-4 on the remaining nodes. In the end you must have the configuration file and the systemd unit file created on every node. 
6. Now it's time to start Patroni. You need the following commands on all nodes but not in parallel. Start with the `111pgsql` first, wait for the service to come to live, and then proceed with the other nodes one-by-one, always waiting for them to sync with the primary node:


    ```{.bash data-prompt="$"}
    sudo systemctl enable --now patroni
    sudo systemctl restart patroni
    ```
   
When Patroni starts, it initializes PostgreSQL (because the service is not currently running and the data directory is empty) following the directives in the bootstrap section of the configuration file. 

7. Check the service to see if there are errors:

    ```{.bash data-prompt="$"}
    sudo journalctl -fu patroni
    ```

    A common error is Patroni complaining about the lack of proper entries in the pg_hba.conf file. If you see such errors, you must manually add or fix the entries in that file and then restart the service.

    Changing the patroni.yml file and restarting the service will not have any effect here because the bootstrap section specifies the configuration to apply when PostgreSQL is first started in the node. It will not repeat the process even if the Patroni configuration file is modified and the service is restarted.

8. Check the cluster. Run the following command on any node:
 
    ```{.bash data-prompt="$"}
    patronictl -c /etc/patroni/patroni.yml list $SCOPE
    ```

    The output resembles the following:

    ```{.text .no-copy}
    + Cluster: cluster_1 (7440127629342136675) --------+----+-----------+
    | Member    | Host           | Role    | State     | TL | Lag in MB |
    +-----------+----------------+---------+-----------+----+-----------+
    | 111pgsql  | 192.168.55.111 | Leader  | running   |  1 |           |
    | 112pgsql  | 192.168.55.112 | Replica | streaming |  1 |         0 |
    | 113pgsql  | 192.168.55.113 | Replica | streaming |  1 |         0 |
    +-----------+----------------+---------+-----------+----+-----------+
    ```

If Patroni has started properly, you should be able to locally connect to a PostgreSQL node using the following command:

```{.bash data-prompt="$"}
sudo psql -U postgres
```

The command output is the following:

```
psql (16)
Type "help" for help.

postgres=#
```

## Configure HAProxy

HAproxy is the load balancer and the single point of entry to your PostgreSQL cluster for client applications. A client application accesses the HAPpoxy URL and sends its read/write requests there. Behind-the-scene, HAProxy routes write requests to the primary node and read requests - to the secondaries in a round-robin fashion so that no secondary instance is unnecessarily loaded. To make this happen, provide different ports in the HAProxy configuration file. In this deployment, writes are routed to port 5000 and reads  - to port 5001

This way, a client application doesn’t know what node in the underlying cluster is the current primary. HAProxy sends connections to a healthy node (as long as there is at least one healthy node available) and ensures that client application requests are never rejected. 

1. Install HAProxy on the `110pgsqlha` node:

    ```{.bash data-prompt="$"}
    sudo apt install percona-haproxy
    ```

2. The HAProxy configuration file path is: `/etc/haproxy/haproxy.cfg`. Specify the following configuration in this file.

    ```
    global
        maxconn 100

    defaults
        log global
        mode tcp
        retries 2
        timeout client 30m
        timeout connect 4s
        timeout server 30m
        timeout check 5s

    listen stats
        mode http
        bind *:7000
        stats enable
        stats uri /

    listen primary
        bind *:5000
        option httpchk /primary 
        http-check expect status 200
        default-server inter 3s fall 3 rise 2 on-marked-down shutdown-sessions
        server 111pgsql 111pgsql:5432 maxconn 100 check port 8008
        server 112pgsql 112pgsql:5432 maxconn 100 check port 8008
        server 113pgsql 113pgsql:5432 maxconn 100 check port 8008

    listen standbys
        balance roundrobin
        bind *:5001
        option httpchk /replica 
        http-check expect status 200
        default-server inter 3s fall 3 rise 2 on-marked-down shutdown-sessions
        server 111pgsql 111pgsql:5432 maxconn 100 check port 8008
        server 112pgsql 112pgsql:5432 maxconn 100 check port 8008
        server 113pgsql 113pgsql:5432 maxconn 100 check port 8008
		
    ```


    HAProxy will use the REST APIs hosted by Patroni to check the health status of each PostgreSQL node and route the requests appropriately. 

3. Restart HAProxy:
    
    ```{.bash data-prompt="$"}
    sudo systemctl restart haproxy
    ```

4. Check the HAProxy logs to see if there are any errors:
   
    ```{.bash data-prompt="$"}
    sudo journalctl -u haproxy.service -n 100 -f
    ```

## Next steps

[Configure pgBackRest](pgbackrest.md){.md-button}
