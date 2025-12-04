# MongoDB Sharding

## Task Overview
Implement a MongoDB sharded cluster with:
- Zone Sharding for `Users` by `region` (`EU`, `USA`, `Asia`)
- Ranged Sharding for `Tweets` by `likes` (0–100, 101–200, 200+)
- Demonstrate correct behavior by shutting down one shard and testing inserts/queries.

---

# Cluster Architecture
- 3 Shard servers (rsShard1, rsShard2, rsShard3)
- 3 Config servers (rsConfig)
- 1 Mongos router

All nodes run inside a Docker network: `mongo-cluster`.

---

# Running the Cluster
## Create `docker-compose.yml`
```yaml
version: '3.9'

networks:
  mongo-cluster:
    driver: bridge

services:
  config-server-01:
    image: mongo:7
    container_name: config-srv-1
    command: ["mongod", "--configsvr", "--replSet", "rsConfig", "--port", "27019", "--bind_ip_all"]
    ports:
      - "27019:27019"
    networks:
      - mongo-cluster

  config-server-02:
    image: mongo:7
    container_name: config-srv-2
    command: ["mongod", "--configsvr", "--replSet", "rsConfig", "--port", "27019", "--bind_ip_all"]
    networks:
      - mongo-cluster

  config-server-03:
    image: mongo:7
    container_name: config-srv-3
    command: ["mongod", "--configsvr", "--replSet", "rsConfig", "--port", "27019", "--bind_ip_all"]
    networks:
      - mongo-cluster

  shard-1:
    image: mongo:7
    container_name: shard-1
    command: ["mongod", "--shardsvr", "--replSet", "rsShard1", "--port", "27018", "--bind_ip_all"]
    ports:
      - "27118:27018"
    networks:
      - mongo-cluster

  shard-2:
    image: mongo:7
    container_name: shard-2
    command: ["mongod", "--shardsvr", "--replSet", "rsShard2", "--port", "27018", "--bind_ip_all"]
    ports:
      - "27218:27018"
    networks:
      - mongo-cluster

  shard-3:
    image: mongo:7
    container_name: shard-3
    command: ["mongod", "--shardsvr", "--replSet", "rsShard3", "--port", "27018", "--bind_ip_all"]
    ports:
      - "27318:27018"
    networks:
      - mongo-cluster

  mongos:
    image: mongo:7
    container_name: mongos
    depends_on:
      - config-server-01
      - config-server-02
      - config-server-03
      - shard-1
      - shard-2
      - shard-3
    command:
      [
        "mongos",
        "--configdb", "rsConfig/config-server-01:27019,config-server-02:27019,config-server-03:27019",
        "--bind_ip_all",
        "--port", "27017"
      ]
    ports:
      - "27017:27017"
    networks:
      - mongo-cluster
```

---

## Start Cluster
- docker compose up -d
- docker compose ps / docker ps

## Initialize Replica Sets
### Config Server RS
- docker exec -it config-server-01 mongosh --port 27019
> rs.initiate({ _id: "rsConfig", configsvr: true, members: [{ _id: 0, host: "config-server-01:27019" },{ _id: 1, host: "config-server-02:27019" },{ _id: 2, host: "config-server-03:27019" }]})

### Shard-1
- docker exec -it shard-1 mongosh --port 27018
> rs.initiate({id: "rsShard1", members: [{ _id: 0, host: "shard-1:27018" }]})

### Shard-2
- docker exec -it shard-2 mongosh --port 27018
> rs.initiate({id: "rsShard2", members: [{ _id: 0, host: "shard-2:27018" }]})

### Shard-3
- docker exec -it shard-3 mongosh --port 27018
> rs.initiate({id: "rsShard3", members: [{ _id: 0, host: "shard-3:27018" }]})

---

## Connect to Mongos
- docker exec -it mongos mongosh --port 27017

## Add Shards
> sh.addShard("rsShard1/shard-1:27018")
> sh.addShard("rsShard2/shard-2:27018")
> sh.addShard("rsShard3/shard-3:27018")

## Enable Sharding for Database
> use labdb
> sh.enableSharding("labdb")

### Users Collection — Zone Sharding
## Create collection & shard key
> db.createCollection("Users")
> sh.shardCollection("labdb.Users", { region: 1, _id: 1 })

## Assign zones
> sh.addShardToZone("rsShard1", "EU")
> sh.addShardToZone("rsShard2", "USA")
> sh.addShardToZone("rsShard3", "Asia")

## Configure ranges
> sh.updateZoneKeyRange("labdb.Users",{ region: "EU", _id: MinKey },{ region: "EU", _id: MaxKey }, "EU")
> sh.updateZoneKeyRange("labdb.Users",{ region: "USA", _id: MinKey },{ region: "USA", _id: MaxKey }, "USA")
> sh.updateZoneKeyRange("labdb.Users",{ region: "Asia", _id: MinKey },{ region: "Asia", _id: MaxKey }, "Asia")

---

### Tweets Collection — Ranged Sharding
## Create & shard collection
> db.createCollection("Tweets")
> sh.shardCollection("labdb.Tweets", { likes: 1 })

## Assign zones
> sh.addShardToZone("rsShard1", "LIKES_0_100")
> sh.addShardToZone("rsShard2", "LIKES_101_200")
> sh.addShardToZone("rsShard3", "LIKES_200_INF")

## Configure ranges
> sh.updateZoneKeyRange("labdb.Tweets",{ likes: 0 },{ likes: 101 }, "LIKES_0_100")
> sh.updateZoneKeyRange("labdb.Tweets",{ likes: 101 },{ likes: 200 }, "LIKES_101_200")
> sh.updateZoneKeyRange("labdb.Tweets",{ likes: 200 },{ likes: MaxKey }, "LIKES_200_INF")

--- 

### Verify Config
> sh.status()

### Testing Sharded Behavior
## Stop Shard2 (USA + likes 101–200)
- docker compose stop shard2

## Should FAIL:
> db.Users.insert({ region: "USA", name: "Bob" })
> db.Tweets.insert({ text: "medium likes", likes: 150 })

## Should WORK:
> db.Users.insert({ region: "EU", name: "Alice" })
> db.Tweets.insert({ text: "low likes", likes: 50 })
> db.Tweets.insert({ text: "viral", likes: 500 })

## Queries (working ranges):
> db.Users.find({ region: "EU" })
> db.Tweets.find({ likes: { $lte: 100 } })

## Queries (offline ranges, should error):
> db.Users.find({ region: "USA" })
> db.Tweets.find({ likes: { $gte: 101, $lte: 200 } })

## Restart Shard2
- docker compose start shard2
