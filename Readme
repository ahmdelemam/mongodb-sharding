## Sharding in MongoDB cluster production-like using docker compose

We will try to make it production like as much as we can
https://www.mongodb.com/docs/manual/core/sharded-cluster-components/#production-configuration

![sharded-cluster-production-architecture](img/sharded-cluster-production-architecture.png)

based on MongoDB docs to have sharded cluster in production we require this

![mongodb-sharded-cluser](img/mongodb-sharded-cluser.png)

## So We need to setup:

- Config servers ReplicaSet (`configserver1`, `configserver2`, `configserver3`) use the `--replSet configReplSet --configserver --bind_ip_all` command to run as a replica set and config servers, binding to all network interfaces, in production you should specify your IPs `--bind_ip localhost,192.51.100.1`
- Shard1 servers ReplicaSet (`shardsvr1`, `shardsvr2`, `shardsvr3`) use the `--replSet r0`
- Shard servers ReplicaSet(`shardsvr4`, `shardsvr5`, `shardsvr6`) use the `--replSet r1`
- The mongos router (`mongos`) uses the `--configdb` flag to connect to the config servers.

## Installation

```bash
docker-compose up
```

Connect to one of configserver members container:

```bash
docker exec -it configserver3 mongosh --port 27017
```

```bash
rs.initiate(
  {
    _id: "configReplSet",
    configserver: true,
    members: [
      { _id: 0, host: "configserver1:27015" },
      { _id: 1, host: "configserver2:27016" },
      { _id: 2, host: "configserver3:27017" },
    ]
  }
)

rs.status()
```

```bash
docker exec -it shardsvr1 mongosh --port 27018
```

```bash
rs.initiate(
  {
    _id: "rs0",
    members: [
      { _id: 0, host: "shardsvr1:27018" },
      { _id: 1, host: "shardsvr2:27019" },
      { _id: 2, host: "shardsvr3:27020" },
    ]
  }
)

rs.status()
```

```bash
docker exec -it shardsvr4 mongosh --port 27022
```

```bash
rs.initiate(
  {
    _id: "rs1",
    members: [
      { _id: 0, host: "shardsvr4:27022" },
      { _id: 1, host: "shardsvr5:27023" },
      { _id: 2, host: "shardsvr6:27024" },
    ]
  }
)

rs.status()
```

```bash
docker exec -it mongos mongosh mongos:27021
```

```bash
sh.addShard("rs0/shardsvr1:27018,shardsvr2:27019,shardsvr3:27020")

sh.addShard("rs1/shardsvr4:27022,shardsvr5:27023,shardsvr6:27024")

db.adminCommand( { listShards: 1 } )

sh.status()
```

Starting in MongoDB 5.1, when starting, restarting or adding a shard server with sh.addShard() the 
Cluster Wide Write Concern (CWWC) must be set.

If the CWWC is not set and the shard is configured such that the default write concern is { w : 1 } the shard server will fail to start or be added and returns an error.

https://www.mongodb.com/docs/manual/reference/command/setDefaultRWConcern/#std-label-set_global_default_write_concern

```bash
docker exec -it mongos mongosh mongos:27021/admin # use admin;

# 2 members required to be majority since we have 3 members in the replica set
db.adminCommand({
  "setDefaultRWConcern" : 1,
  "defaultWriteConcern" : {
    "w" : 2
  },
  "defaultReadConcern" : { "level" : "majority" }
})
```

The default range size for a sharded cluster is 128 megabytes. 
The allowed size is between 1 and 1024 megabytes, inclusive.

To modify the range size, use the following procedure:

Connect to any mongos in the cluster using 
mongosh.

Switch to the Config Database to store the global range size configuration value: <sizeInMB>
https://www.mongodb.com/docs/manual/tutorial/modify-chunk-size-in-sharded-cluster/#std-label-tutorial-modifying-range-size

```bash
docker exec -it mongos mongosh mongos:27021/config # use config;

db.settings.updateOne(
   { _id: "chunksize" },
   { $set: { _id: "chunksize", value: 1 } },
   { upsert: true }
)
```


Sharding and Indexes
If the collection already contains data, you must create an index that supports the shard key before sharding the collection.
If the collection is empty, MongoDB creates the index as part of sh.shardCollection().

```bash
db.users.createIndex({"username" : 1}) # if needed

sh.shardCollection("accounts.users", {"username" : "hashed"})

sh.shardCollection("<database>.<collection>", { <shard key field> : "hashed" } )
sh.shardCollection("<database>.<collection>", { <shard key field> : 1, <shard key field2> : 1, ... } )

sh.status()
```

We will find there are 2  empty chunks already created in each replicaSet.

let's add some data!!

```bash
use accounts

var users = [];
for (var i = 0; i < 100; i++) {
    users.push({ "username": "user" + i, "email": "user" + i + "@gmail.com", "created_at": new Date() });
}

db.users.insertMany(users);

db.users.count();

db.users.find().explain();

db.users.find({username: "user1"}).explain();
```