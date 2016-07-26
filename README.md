#MongoDB 3.2.9 Replica Sets on AWS EC2 with arbiter
A MongoDB replica set provides a mechanism to allow for a reliable database services. The basic replica set consists of three servers, a primary, a secondary and an arbitrator. The primary and secondary both hold a copy of the data. The arbitrator is normally a low spec server which just monitors the other servers and help with the failover process. In production, there can be more than three servers.

To setup mongo as a replica set on Amazon Web Services EC2 you need to first setup a security group with ssh on port 22 and mongodb on port 27017. You then need to create three servers. Select Centos 7 and a micro (or bigger depending on your database size, ideally you should have enough memory to match your database size) instance for the primary and secondary and a nano instance for the arbitrator.

Very Important!!!!  Please disable SElinux on Centos or Redhat Versions. This will silent not allowed to change context the server. so your mongod service wont start if you pointing to different path and also it will create multiple issues like authenctiaon etc.


##Adjust the File System on each Server
The operating system by default will update the last access time on a file. In a high data throughput database application this is overhead which will slow down the database. Therefore, disable this feature edit the fstabs file using:

    sudo vim /etc/fstab
    
Add ```noatime``` directly after ```defaults,```.

    LABEL=cloudimg-rootfs   /        ext4   defaults,noatime        0 0

##Setup MongoDB on each Server

echo " [mongodb-org-3.2]
name=MongoDB Repository
baseurl=https://repo.mongodb.org/yum/redhat/$releasever/mongodb-org/3.2/x86_64/
gpgcheck=1
enabled=1
gpgkey=https://www.mongodb.org/static/pgp/server-3.2.asc " >> /etc/yum.repos.d/mongod.repo

## Install Mongodb binaries from Yum

yum -y install mongodb-org mongodb-org-server 

    
#Stop the mongod server for a while.

    sudo service mongod stop

##Modify the hosts on each Server
Modify the hosts file on each server. Obviously on mongo1 the 127.0.0.1 will point at localhost mongo1.example.com

    sudo vim  /etc/hosts
    
    127.0.0.1           localhost mongo0.example.com
    52.51.12.62         mongo0.example.com
    52.18.54.237        mongo1.example.com
    52.40.234.42        mongo2.example.com

##Modify the hostname on each Server
To make it easier, it is best to modify the hostname to make the servers easier to reference. First set the current hostname.

    sudo hostname mongo0.example.com
    
Then update the hostname file to set the server name permanently.

    sudo nano /etc/hostname

Set the hostname in the file to:

    mongo0.example.com

##Modify the Mongo Configuration
You need to modify the mongo configuration ```/etc/mongod.conf``` to first remove or comment out the ```bind_ip```. This will tell mongo to listen on all interfaces.

    # mongod.conf                                                               
    
    # for documentation of all options, see:                                    
    #   http://docs.mongodb.org/manual/reference/configuration-options/         
      
    # Where and how to store data.                                              
    storage:                                                                    
      dbPath: /var/lib/mongodb                                                  
      journal:                                                                  
        enabled: true                                                           
    #  engine:                                                                  
    #  mmapv1:                                                                  
    #  wiredTiger:                                                              
     
    # where to write logging data.                                              
    systemLog:                                                                  
      destination: file                                                         
      logAppend: true                                                           
      path: /var/log/mongodb/mongod.log                                         
     
    # network interfaces                                                        
    net:                   
      #bindIp: 127.0.0.1 - Important to comment this line
      port: 27017                                                               
  
    #processManagement:                                                         
       
    #security:                                                                  
       
    #operationProfiling:                                                        
      
    #replication:                                                               
    replication:                                                                
       oplogSizeMB: 1                                                           
       replSetName: rs0                                                         
    
    #sharding:                                                                  
    
    ## Enterprise-Only Options:                                                 
    
    #auditLog:                                                                  
      
    #snmp:   
    
##Initiate the Replication Set
Start the mongo shell on ```mongo0```:

    mongo
    
Add the first server which will become the Primary to the replica set using:

    rs.initiate()
    
This will create the initial replica set configuration. You can then add in the second server which will become the Secondary.

    rs.add("mongo1.example.com:27017")
    
You can then add in the third server which will be the Arbitrator. Note to make this an Arbitrator you need to specify ```true``` when you add it to the set.

    rs.add("mongo2.example.com:27017",true)
    
##Replica Set Config
You can view the Reclica Set Config using:

    rs.config()
    
You can also assign this to a variable using ```var conf = rs.config()```. The configuration should look like:

    {
        "_id" : "rs0",
        "version" : 5,
        "protocolVersion" : NumberLong(1),
        "members" : [
                {
                        "_id" : 0,
                        "host" : "mongo0.example.com:27017",
                        "arbiterOnly" : false,
                        "buildIndexes" : true,
                        "hidden" : false,
                        "priority" : 1,
                        "tags" : {

                        },
                        "slaveDelay" : NumberLong(0),
                        "votes" : 1
                },
                {
                        "_id" : 1,
                        "host" : "mongo1.example.com:27017",
                        "arbiterOnly" : false,
                        "buildIndexes" : true,
                        "hidden" : false,
                        "priority" : 1,
                        "tags" : {

                        },
                        "slaveDelay" : NumberLong(0),
                        "votes" : 1
                },
                {
                        "_id" : 2,
                        "host" : "mongo2.example.com:27017",
                        "arbiterOnly" : true,
                        "buildIndexes" : true,
                        "hidden" : false,
                        "priority" : 1,
                        "tags" : {

                        },
                        "slaveDelay" : NumberLong(0),
                        "votes" : 1
                }
        ],
        "settings" : {
                "chainingAllowed" : true,
                "heartbeatIntervalMillis" : 2000,
                "heartbeatTimeoutSecs" : 10,
                "electionTimeoutMillis" : 10000,
                "getLastErrorModes" : {

                },
                "getLastErrorDefaults" : {
                        "w" : 1,
                        "wtimeout" : 0
                }
        }
    }

###Replica Set Status
To check on the status of a replica set just use:

    rs.status()
    
The status should look like:

    {
        "set" : "rs0",
        "date" : ISODate("2016-02-28T19:48:13.094Z"),
        "myState" : 1,
        "term" : NumberLong(3),
        "heartbeatIntervalMillis" : NumberLong(2000),
        "members" : [
                {
                        "_id" : 0,
                        "name" : "mongo0.example.com:27017",
                        "health" : 1,
                        "state" : 1,
                        "stateStr" : "PRIMARY",
                        "uptime" : 7704,
                        "optime" : {
                                "ts" : Timestamp(1456688888, 1),
                                "t" : NumberLong(3)
                        },
                        "optimeDate" : ISODate("2016-02-28T19:48:08Z"),
                        "electionTime" : Timestamp(1456681189, 1),
                        "electionDate" : ISODate("2016-02-28T17:39:49Z"),
                        "configVersion" : 5,
                        "self" : true
                },
                {
                        "_id" : 1,
                        "name" : "mongo1.example.com:27017",
                        "health" : 1,
                        "state" : 2,
                        "stateStr" : "SECONDARY",
                        "uptime" : 7629,
                        "optime" : {
                                "ts" : Timestamp(1456688888, 1),
                                "t" : NumberLong(3)
                        },
                        "optimeDate" : ISODate("2016-02-28T19:48:08Z"),
                        "lastHeartbeat" : ISODate("2016-02-28T19:48:12.120Z"),
                        "lastHeartbeatRecv" : ISODate("2016-02-28T19:48:12.124Z"),
                        "pingMs" : NumberLong(0),
                        "syncingTo" : "mongo0.example.com:27017",
                        "configVersion" : 5
                },
                {
                        "_id" : 2,
                        "name" : "mongo2.example.com:27017",
                        "health" : 1,
                        "state" : 7,
                        "stateStr" : "ARBITER",
                        "uptime" : 2,
                        "lastHeartbeat" : ISODate("2016-02-28T19:48:12.124Z"),
                        "lastHeartbeatRecv" : ISODate("2016-02-28T19:48:10.120Z"),
                        "pingMs" : NumberLong(0),
                        "configVersion" : 5
                }
        ],
        "ok" : 1
    }

###Testing the Replica Set
To test that it is working correctly you need to add a document to the Primary.

    db.foo.save({_id:1, value:'hello world'})

Check that the data is saved on the primary:

    db.foo.find()

This will return:

    { "_id" : 1, "value" : "hello world" }

Then log onto the secondary. This will have a prompt of ```demo:SECONDARY>```.

    db.foo.find()

This will return:

    { "_id" : 1, "value" : "hello world" }


###Error not master and slaveOk=false
You may get an initial error ```error: { "$err" : "not master and slaveOk=false", "code" : 13435 }``` when you are trying to run ```db.foo.find()``` on the Secondary. This is because the Secondary (ie the Slave) is not setup to perform reads. This can be enabled using the following command:

    db.setSlaveOk()

You should then be able to access the data.

###Updating the configuration
You can remove a server from the Replica Set using:

    rs.remove("mongo0.example.com")
    
As mentioned before you can assign the configuration to a variable ```var conf = rs.config()```. You can then edit this configuration and get mongo to reload the configuration using:

    rs.reconfig(conf)

Infact you can create the configuration from the start using:

    rs.initiate(conf)
    
Provided that you have already assigned conf to the configuration from a copy of the configuration opject.

##Failover
If a Primary fails then an election is held and the Secondary and Arbitrator vote for the Secondary (since this is the only server left holding data) and since these two represent over 50% (ie 66%) of the replica set then the election succeeds and the Secondary becomes the new Primary. Mongo requires a Primary to be able to accept write requests. If the Arbitrator is unavailable too then the Secondary will not become the Primary since only 33% of the replica set members can vote. In this case the mongodb will still accept read requests but cannot do writes.

If the old Primary comes back on line it will become the new Secondary as both servers have equal priority. You can change this by setting the priority of the servers. So if the old Primary comes back on line it will become the Primary again.

    var conf - rs.config()
    
The Primary is the first in the members array.

    conf.members[0].priority = 10

Apply the new configuration. This needs to be done from the current Primary.

    rs.reconfig(conf)
    
If you don't have a Primary you can force the replica set to use the new configuration but you should not do this with a healty replica set.

    rs.reconfig(conf,{force: true})
    
##SetDown
There are times when you need to do work on the primary server in a replica set. To do this you need to take the server out of the replica set temporarly. To do this you can use the ```rs.stepDown(seconds)```` command. For example, to step down a server for 3 minutes:

    rs.stepDown(60*3)
  
##Freeze
Sometimes you may need to work on a Secondary server and you need to ensure that it does not become a Primary while you are working on it. To do this you can issue a ```rs.freeze(seconds)``` to temporarly stop the Secondary from becoming a Primary.

##Hidden Server
You can also create a Hidden server. This is a server that replicates with the other members of the replica set but which is cannot be seen by the Application (hidden = true) and which can never become a Primary server (ie priority = 0). It can however, vote in an election of a new Primary. This is useful for reporting for example or it can be used to bring a new server into the replica set and then make it available at a later point (such as when it has had a chance to sync up all the data). To set a server (ie Abritrator) to hidden:

    var conf = rs.config()
    conf.members[2].priority = 0
    conf.members[2].hidden = true
    rs.reconfig(conf)
    
##Durability
Mongo supports different levels of durability. These are called "write concerns".

###Acknowledged write concern
This means that mongo updates its in memory view of the data but does not indicate that the data has been written.

###Unacknowledged write concern
This is the fastest write you can perform as it does not even require that it has been logged in memory. Mongo will only raise an error the command is not accepted.

    db.demo.insert({x: 'hello'},{writeConcern: {w:0}})
    WriteResult({ })

Both Acknowledged and Unacknowledged are fast but risky but can be useful where the loss of small amounts of data is not a big concern such as page logging or large volume sensor readings etc.

###Journaled write concern
Mongo will hold off returning until the data has been written to the jounal. This is the default for Mongo.

    db.demo.insert({x: 'hello'},{writeConcern: {j:true}})
    WriteResult({ "nInserted": 1 })
    

###[w2|w3..|majority] write concern
You tell Mongo with write to a certain number of members or the majority of members. First lets ask for acknowledgement from one server.

    db.demo.insert({x: 'hello'},{writeConcern: {w:1}})
    WriteResult({ "nInserted": 1 })
   
Then two servers:

    db.demo.insert({x: 'hello'},{writeConcern: {w:2}})
    WriteResult({ "nInserted": 1 })    

Acknowledged by two servers and written to the journal:

    db.demo.insert({x: 'hello'},{writeConcern: {w:2, j:true}})
    WriteResult({ "nInserted": 1 })    


Then a majority:
    
    db.demo.insert({x: 'hello'},{writeConcern: {w:'majority'}})
    WriteResult({ "nInserted": 1 }) 
    
##Write Timeouts
It is possible to set a timeout on the write so that if there is a problem that means that the write does not happen in an expected time an error is passed back to the client so it can take alternative action.

    db.demo.insert({x: 'hello'},{writeConcern: {w:3, wtimeout: 2000}})
    WriteResult({ 
            "nInserted": 1,
            "writeConcernError": {
                     "code": 64,
                     "errInfo": {
                             "wtimeout": true
                     }
                     "errmsg": "waiting for replication timed out"
            }
    }) 

In this case the document was saved but it was not acknowleged by three servers in the 2 second (2000 miliseconds) time period.



##############  Secure your mongodb with authorization and authentication##################

Simple steps :

(1) Make sure you disabled selinux 
(2) Before enable authentication need to create users to login on mongodb. follow below and modify the permissions as you like.
    
   create 2 users like these: 
   --------------------------------------------------- 
  use admin
  db.createUser(
    {
    user: "siteUserAdmin",
    pwd: "password",
    roles: [ { role: "userAdminAnyDatabase", db: "admin" } ]
     }
     )
    ----------------------------------------------------------
  use admin
  db.createUser(
    {
    user: "siteRootAdmin",
    pwd: "password",
    roles: [ { role: "root", db: "admin" } ]
     }
     )
    ----------------------------------------------------------

(3) generate the keyfile  as below 
    openssl rand -base64 741 > mongodb-keyfile
    chmod 600 mongodb-keyfile
(4) copy these keys to all mongo servers and change the permissions accordingly. use whatever method like to copy. i prefer sftp command
(5) stop other secondary hosts and arbiter first then primary.
(6) open mongo.conf from /etc/mongod.conf

    Edit as below that point newly generated key.
  
    security:

   authorization: "enabled"
   keyFile: /secure/keys/mongodb-keyfile

(4) do the same procedure on all hosts 
(5) start the mongod instance on primary then rest all.

##############  All done ################  

You can tail the logs to check any issues  tail /var/log/mongo/mongod.log
