
# Mongo DB Installation using Ansible.

## This Repository `ansible-mongodb-deploy` for getting started with Ansible.

This is a quick and dirty deployment of mongodb which I created to help a friend get started with `ansible`, since he was familiar with mongdb so we create a ansible playbook to get the directory setup on the cluster nodes. 

## Command to get started.

Installing `ansible`

    sudo yum install ansible

Execute below command in `mongodb-deployment-hmt` directory, adhoc command to execute on all servers.

    ansible -i hosts all -m ping -u root --ask-pass
    ansible -i hosts all -m shell -a 'ls' -u root --ask-pass
    ansible -i hosts all -m shell -a 'cat /etc/hosts' -u root --ask-pass
    ansible -i hosts all -m shell -a 'ls -l /lognbin/' -u root --ask-pass

Executing `ansible-playbook`

    ansible-playbook -i hosts site.yml -u root --ask-pass


### Created from the below snippet, to get started.


First the servers.

    /* Assuming Mongo Cluster has to be done on below 3 Nodes*/
    192.168.62.149 host_name=mongodb1.cluster1.ahmed.com
    192.168.62.157 host_name=mongodb2.cluster1.ahmed.com
    192.168.62.158 host_name=mongodb3.cluster1.ahmed.com
    192.168.62.159 host_name=mongodb4.cluster1.ahmed.com
    192.168.62.160 host_name=mongodb5.cluster1.ahmed.com

    
Get a user on all the nodes, we are creating `mongoadmin`
 
    /* We can also add a Step to create a non root user in group (mongoadmin or mongodb) below is example of creating a username and group as mongodb*/
    --------------PreReqisits-------------------------------
    root@ahmed-server-1-VM7:~# groupadd mongodb
    root@ahmed-server-1-VM7:~# useradd mongodb -g mongodb
    root@ahmed-server-1-VM7:~# passwd mongodb
    Enter new UNIX password:
    Retype new UNIX password:
    passwd: password updated successfully

Creating directories for the data, and give them permissions.    
    
    root@ahmed-server-1-VM7:~# mkdir -p /lognbin /dataDB /journalidx


    chown -R mongoadmin:mongoadmin /lognbin
    chown -R mongoadmin:mongoadmin /dataDB
    chown -R mongoadmin:mongoadmin /journalidx

    
Disable `transparent_hugepage`.     
    
    /*OS Level configurations for Ubuntu below has to be done from root user or sudo user */
    cp /etc/rc.local /etc/rc.local_back      			--(taking backup before changes optional)
    nano /etc/rc.local 									--Edit the file in # prompt and paste the below lines in last,it can be above exit 0,needs reboot

    if test -f /sys/kernel/mm/transparent_hugepage/enabled; then
       echo never > /sys/kernel/mm/transparent_hugepage/enabled
    fi
    if test -f /sys/kernel/mm/transparent_hugepage/defrag; then
       echo never > /sys/kernel/mm/transparent_hugepage/defrag
    fi	


Setting `nproc` for our user.

    root@mongo-server-1:~#
    mongoadmin@pilot-mongo-01:~# nano /etc/security/limits.d/90-mongodb-nproc.conf  (if this file not exists we have to create new)
    mongoadmin   soft    nofile   64000
    mongoadmin   hard    nofile   64000
    root         soft    nofile   64000
    root         hard    nofile   64000
    /*No Need to reboot just reconnect it will work */

    
Creating more directories.
    
    --Mongo Installation 		Ensure you are not is root user 
    mongoadmin@mongo-server-1:~$
    mkdir /lognbin/mglog
    mkdir /dataDB/db
    
Extracting `mongodb` from archives.
    
    cd /lognbin
    wget https://fastdl.mongodb.org/linux/mongodb-linux-x86_64-ubuntu1604-3.2.12.tgz
    tar xzf mongodb-linux-x86_64-ubuntu1604-3.2.12.tgz
    mv mongodb-linux-x86_64-ubuntu1604-3.2.12 mgbin 			(this is optional to keep binary folder simple)
  
Copy to all servers.
  
    --Copying the untar folder to other Nodes,-r used due to folder copy (No Need to download again)	

    scp -r /lognbin/mgbin mongoadmin@mongdb2.cluster1.ahmedinc.com:/lognbin
    scp -r /lognbin/mgbin mongoadmin@mongdb3.cluster1.ahmedinc.com:/lognbin
    scp -r /lognbin/mgbin mongoadmin@mongdb4.cluster1.ahmedinc.com:/lognbin
    scp -r /lognbin/mgbin mongoadmin@mongdb5.cluster1.ahmedinc.com:/lognbin

Sync all configurations.    

        --(pulling files from mongdb1.cluster1.ahmedinc.com)
    scp mongoadmin@mongdb1.cluster1.ahmedinc.com:/lognbin/mauth.conf /lognbin
    scp mongoadmin@mongdb1.cluster1.ahmedinc.com:/lognbin/noauth.conf /lognbin
    scp mongoadmin@mongdb1.cluster1.ahmedinc.com:/lognbin/HAkey /lognbin

    /*
    Now we are in 10.168.110.71
    Edit the ReplicaSet name as "ahmedset123" in both mauth.conf and noauth.conf file and copy to other Nodes as below
    All 3 files together*/

    scp mauth.conf noauth.conf HAkey mongoadmin@mongdb2.cluster1.ahmedinc.com:/lognbin
    scp mauth.conf noauth.conf HAkey mongoadmin@mongdb3.cluster1.ahmedinc.com:/lognbin
    scp mauth.conf noauth.conf HAkey mongoadmin@mongdb4.cluster1.ahmedinc.com:/lognbin
    scp mauth.conf noauth.conf HAkey mongoadmin@mongdb5.cluster1.ahmedinc.com:/lognbin

Creating configuration files `mauth.conf` and `noauth.conf`.    
    
    /* below is the format of mauth.conf and noauth.conf where we need to change the replicaset name from "ahmedz1" to "ahmedset123"
       File mauth.conf and noauth.conf format will be same except noauth.conf will not have security: section*/
    storage:
       dbPath: "/dataDB/db"
       directoryPerDB: true
       wiredTiger:
          engineConfig:
             directoryForIndexes: true
       journal:
            enabled: true
    systemLog:
       destination: file
       path: "/lognbin/mglog/mongodb.log"
    processManagement:
       fork: true
    net:
       port: 27127
    replication:
       oplogSizeMB: 35840
       replSetName: "ahmedset123"
    security:
       keyFile: "/lognbin/HAkey"
       authorization: enabled
       
Creating the `HAkey` file required in the configuration above.
       
    --Creating Key File-----------   (As we have copied from other machine no need of below 2 Lines)
    openssl rand -base64 755 > HAkey
    chmod 400 HAkey   -- this is very important else it will give Error of permission " permissions on /lognbin/HAkey are too open"
        

Starting Services and setting up mongodb. [NOTE : THIS NEEDS TO BE IMPLEMENTED]        
        
    /*--------Starting mongo without authentication----------*/
    mongod -f /lognbin/noauth.conf

    /* connecting to Mongo Shell */
    mongo mongodb1.cluster1.ahmed.com:27127            (Without authentication)

    use admin
    rs.initiate({_id:"ahmedset123",members:[{_id:0,host:"mongodb1.cluster1.ahmed.com:27127",tags: { "Site": "ahmedz1", "use": "Live"  }}]})

    --Ensure in use admin
    db.runCommand( { createUser : "DBADMIN", pwd: "dbadmin@123", roles: [ { role: "root", db: "admin" }, { role: "restore", db: "admin" } ] } )

    --Now ShutDown and restart the Mongo with Authentication enabled
    --As Mongo started with out authentication we have to kill the process

    /*Start the mongo in all Nodes with Authentication enabled*/
    mongod -f /lognbin/mauth.conf    


    /*Login to mongodb1 shell with below command */
    mongo mongodb1.cluster1.ahmed.com:27127/admin -u DBADMIN -p dbadmin123 	(With Credentials)

    /* Assuming the login Node is PRIMARY */
    rs.add( { host: "mongodb2.cluster1.ahmed.com:27127", tags: { "Site": "ahmedz1", "use": "Live"  } } )
    rs.add( { host: "mongodb3.cluster1.ahmed.com:27127", tags: { "Site": "ahmedz2", "use": "Live"  } } )
    rs.add( { host: "mongodb4.cluster1.ahmed.com:27127", tags: { "Site": "ahmedz2", "use": "Live"  } } )
    rs.add( { host: "mongodb5.cluster1.ahmed.com:27127", tags: { "Site": "ahmedz2", "use": "Delayed"  }, priority: 0,slaveDelay: 10800 } )
    rs.add( { host: "mongodb6.cluster1.ahmed.com:27127", tags: { "Site": "ahmedz1", "use": "Export"  }, priority: 0,votes: 0 } )
