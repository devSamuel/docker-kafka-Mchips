# Kafka KRaft Cluster Architecture

## Cluster Overview

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         Kafka KRaft Cluster                              │
│                    Cluster ID: HacAgFbaRJ6PuDyphbLNFA                   │
└─────────────────────────────────────────────────────────────────────────┘

                    ┌──────────────────────────────┐
                    │   Controller Quorum (3/3)    │
                    │    (Metadata Management)      │
                    └──────────────────────────────┘
                                 │
        ┌────────────────────────┼────────────────────────┐
        │                        │                        │
        ▼                        ▼                        ▼
┌───────────────┐       ┌───────────────┐       ┌───────────────┐
│ Controller-1  │◄─────►│ Controller-2  │◄─────►│ Controller-3  │
│   Node ID: 1  │       │   Node ID: 2  │       │   Node ID: 3  │
│   Port: 9093  │       │   Port: 9093  │       │   Port: 9093  │
│  (Ephemeral)  │       │  (Ephemeral)  │       │  (Ephemeral)  │
└───────┬───────┘       └───────┬───────┘       └───────┬───────┘
        │                       │                       │
        └───────────────────────┼───────────────────────┘
                                │
                    Metadata Replication
                    (Raft Consensus)
                                │
        ┌───────────────────────┼───────────────────────┐
        │                       │                       │
        ▼                       ▼                       ▼
┌───────────────┐       ┌───────────────┐       ┌───────────────┐       ┌───────────────┐
│   Broker 1    │◄─────►│   Broker 2    │◄─────►│   Broker 3    │◄─────►│   Broker 4    │
│  Node ID: 6   │       │  Node ID: 5   │       │  Node ID: 4   │       │  Node ID: 7   │
│ Host:9093→9092│       │ Host:9092→9092│       │ Host:9091→9092│       │ Host:9090→9092│
│ Internal:9092 │       │ Internal:9092 │       │ Internal:9092 │       │ Internal:9092 │
│               │       │               │       │               │       │               │
│  Persistent   │       │  Persistent   │       │  Persistent   │       │  Persistent   │
│    Volume     │       │    Volume     │       │    Volume     │       │    Volume     │
│ broker1-data  │       │ broker2-data  │       │ broker3-data  │       │ broker4-data  │
└───────┬───────┘       └───────┬───────┘       └───────┬───────┘       └───────┬───────┘
        │                       │                       │                       │
        └───────────────────────┼───────────────────────┼───────────────────────┘
                                │    Data Replication    │
                                │   (Topic Partitions)   │
                                │                        │
                                ▼                        │
                        ┌───────────────┐                │
                        │   Kafka UI    │                │
                        │  Port: 8081   │◄───────────────┘
                        │               │
                        │  Monitoring   │
                        │  Management   │
                        └───────────────┘
                                │
                                ▼
                        Your Applications
                     (Producers/Consumers)
```
## How to use it
We need to run the containers as a swarm orchestrator, then deploy the stack. Well first you need to place in your terminal  and run these commands 

docker swarm init 

docker stack deploy -c compose.yml sample-kafka-stack 

### Getting container id 
docker ps --filter "label=com.docker.swarm.service.name=sample-kafka-stack_broker1" --no-trunc --format "{{.ID}}" 

or just go directly to your docker desktop and get the container id from the listed brokers. 

### Creating topic 
You can either open the terminal directly in any broker container or get your container id and run  
1st option: /opt/bitnami/kafka/bin/kafka-topics.sh --bootstrap-server broker1:9092 --create --topic new-orders --partitions 16 --replication-factor 2 

2nd option: docker exec -it "container_id" /opt/bitnami/kafka/bin/kafka-topics.sh --bootstrap-server broker1:9092 --create --topic new-orders --partitions 16 --replication-factor 2 


### Connecting a consumer 
docker exec -it  "container_id" /opt/bitnami/kafka/bin/kafka-console-consumer.sh --bootstrap-server broker1:9092 --topic  new-orders 


### Connecting a producer 
docker exec -it  "container_id" /opt/bitnami/kafka/bin/kafka-console-producer.sh --bootstrap-server broker1:9092 --topic new-orders 

Type any message and you'll see it in the consumer terminal 


## Component Details

### Controllers (Quorum - 3 nodes)
- **Purpose**: Manage cluster metadata, leader election, partition assignments
- **Mode**: KRaft (Kafka Raft - no ZooKeeper needed)
- **Quorum**: 3 controllers form a Raft consensus group
- **Communication**: Internal port 9093 (CONTROLLER protocol)
- **Storage**: Ephemeral (rebuilt from broker metadata on restart)
- **Resources**:
  - CPU: 0.5-3.5 cores
  - Memory: 1GB-6GB

**Controller Details:**
```
Controller-1: kafka-controller-1:9093 (Node ID: 1)
Controller-2: kafka-controller-2:9093 (Node ID: 2)
Controller-3: kafka-controller-3:9093 (Node ID: 3)
```

**Quorum Voters Configuration:**
```
1@kafka-controller-1:9093,2@kafka-controller-2:9093,3@kafka-controller-3:9093
```

### Brokers (Data Plane - 4 nodes)

#### Broker 1
- **Node ID**: 6
- **External Port**: localhost:9093 → container:9092
- **Internal**: broker1:9092
- **Volume**: broker1-data (persistent)
- **Resources**: CPU: 0.5-3.5 cores, Memory: 1GB-6GB

#### Broker 2
- **Node ID**: 5
- **External Port**: localhost:9092 → container:9092
- **Internal**: broker2:9092
- **Volume**: broker2-data (persistent)
- **Resources**: CPU: 0.5-3.5 cores, Memory: 1GB-6GB

#### Broker 3
- **Node ID**: 4
- **External Port**: localhost:9091 → container:9092
- **Internal**: broker3:9092
- **Volume**: broker3-data (persistent)
- **Resources**: CPU: 0.5-4.5 cores, Memory: 1GB-6GB

#### Broker 4
- **Node ID**: 7
- **External Port**: localhost:9090 → container:9092
- **Internal**: broker4:9092
- **Volume**: broker4-data (persistent)
- **Resources**: CPU: 0.5-3.5 cores, Memory: 1GB-6GB

## Communication Flows

### 1. Controller Quorum Communication
```
Controller-1 ◄──── Raft Protocol ────► Controller-2
     ▲                                      ▲
     │                                      │
     │          Raft Protocol               │
     │                                      │
     └──────────────────►Controller-3◄──────┘
```
- **Protocol**: Raft consensus (leader election, log replication)
- **Port**: 9093 (CONTROLLER listener)
- **Purpose**: Maintain consistent cluster metadata

### 2. Broker-to-Controller Communication
```
Broker 1, 2, 3, 4
       │
       │ Fetch Metadata
       │ Register Broker
       │ Report Status
       ▼
Controller Quorum
```
- Brokers connect to all controllers
- Metadata fetching (topics, partitions, ISR)
- Broker registration and heartbeats

### 3. Broker-to-Broker Communication
```
Broker 1 ◄──── Replication ────► Broker 2
     ▲                                ▲
     │                                │
     │         Replication            │
     │                                │
Broker 4 ◄──── Replication ────► Broker 3
```
- **Protocol**: PLAINTEXT (inter-broker communication)
- **Port**: 9092 (internal)
- **Purpose**: Partition replication, leader/follower sync

### 4. Client-to-Broker Communication
```
Producers/Consumers
       │
       │ Produce/Consume Messages
       │ localhost:9090-9093
       ▼
  Broker 1, 2, 3, 4
```
- **External Ports**:
  - Broker 1: localhost:9093
  - Broker 2: localhost:9092
  - Broker 3: localhost:9091
  - Broker 4: localhost:9090

## Data Flow Example

### Publishing a Message
```
1. Producer → Broker (any) → Fetch Metadata from Controllers
2. Producer → Partition Leader Broker
3. Leader Broker → Follower Brokers (replication)
4. Leader Broker → Acknowledge to Producer
```

### Consuming a Message
```
1. Consumer → Broker (any) → Fetch Metadata from Controllers
2. Consumer → Subscribe to Topic
3. Consumer → Fetch from Partition Leader(s)
4. Consumer → Commit Offsets
```

## Fault Tolerance

### Controller Quorum (3 nodes)
- **Fault Tolerance**: Can tolerate 1 controller failure
- **Quorum Requirement**: 2/3 controllers must be available
- **Leader Election**: Automatic via Raft consensus

### Broker Cluster (4 nodes)
- **Replication Factor**: Configure per topic (e.g., 3)
- **ISR (In-Sync Replicas)**: Maintains data consistency
- **Partition Distribution**: Spread across brokers

## Access Points

### For Applications
- **Bootstrap Servers**: `localhost:9090,localhost:9091,localhost:9092,localhost:9093`
- **Internal DNS**: `broker1:9092,broker2:9092,broker3:9092,broker4:9092`

### For Management
- **Kafka UI**: http://localhost:8081
  - Connected to: `broker1:9092,broker2:9092,broker3:9092,broker4:9092`
  - View topics, partitions, consumer groups, brokers

## Network Architecture

```
┌─────────────────────────────────────────────────────┐
│              Docker Network: kafka-net              │
│                                                     │
│  ┌─────────────┐  ┌─────────────┐  ┌────────────┐  │
│  │Controllers  │  │  Brokers    │  │ Kafka UI   │  │
│  │  (3 nodes)  │  │  (4 nodes)  │  │            │  │
│  └─────────────┘  └─────────────┘  └────────────┘  │
└─────────────────────────────────────────────────────┘
         │                   │                │
         │                   │                │
    Port 9093          Ports 9090-9093   Port 8081
         │                   │                │
         └───────────────────┴────────────────┘
                             │
                      Host Network
                             │
                    Your Applications
```

## Resource Summary

| Component      | Count | CPU (Reserved/Limit) | Memory (Reserved/Limit) | Storage         |
|----------------|-------|----------------------|-------------------------|-----------------|
| Controllers    | 3     | 0.5 - 3.5 cores      | 1GB - 6GB               | Ephemeral       |
| Brokers        | 4     | 0.5 - 4.5 cores      | 1GB - 6GB               | Persistent      |
| Kafka UI       | 1     | Default              | Default                 | None            |
| **Total**      | **8** | **4 - 30.5 cores**   | **8GB - 48GB**          | 4 volumes       |

## Key Features

✅ **No ZooKeeper** - Uses KRaft mode (Kafka Raft)
✅ **High Availability** - 3 controllers, 4 brokers
✅ **Data Persistence** - Broker data survives restarts
✅ **Web UI** - Easy cluster management
✅ **Scalable** - Can add more brokers easily
✅ **Resource Managed** - CPU and memory limits set
