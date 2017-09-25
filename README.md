# Kafka Partition Rebalance Tool

## Introduction
Kafka is not aware of the cluster topology (not rack aware) and hence partitions are susceptible to data loss or unavailability in the event of faults or updates. 

This tool generates a reassignment plan that has two goals:
- Redistribute the replicas of partitions of a topic across brokers in a manner such that all replicas of a partition are in separate Update Domains (UDs) & Fault Domains (FDs).
- Balance the leader load across the cluster - The number of leaders assigned to each broker is more or less the same. 

Once the reasignment plan is generated the user can execute this plan. Upon execution, the tool updates the zookeeper path '/admin/reassign_partitions' with the list of topic partitions and (if specified in the Json file) the list of their new assigned replicas. The controller listens to the path above. When a data change update is triggered, the controller reads the list of topic partitions and their assigned replicas from zookeeper. The control handles starting new replicas in RAR and waititing until the new replicas are in sync with the leader. At this point, the old replicas are stopped and the partition is removed from the '/admin/reassignpartitions' path. Note that these steps are async operations.

This tool is best stuitable for executing on 
- New cluster
- When a new topic/partition is created
- Cluster is scaled up

Note for execution on existing clusters:
When executing on a cluster with large data sizes, there will be a performance degradation while reassignment is taking place, due to data movement. On large clusters, rebalance can take several hours. In such scenarios, it is recommended to execute rebalance by steps (by topic or by partitions of a topic).

## Prequisites
The script relies on various python modules. If not installed already, you must manually install these prior to execution:
```
sudo apt-get update
sudo apt-get install libffi-dev libssl-dev
sudo pip install --upgrade requests[security] PyOpenSSL ndg-httpsclient pyasn1 kazoo retry pexpect
```

## How to use
Copy the file to /usr/hdp/current/kafka-broker/bin, and run it as ROOT.

```
usage: sudo rebalance_rackaware.py [-h] [-topics TOPICS [TOPICS ...]] [-execute]
                        [-verify] [-computeStorageCost] [-username USERNAME]
                        [-password PASSWORD] [-force]

Rebalance Kafka Replicas.

optional arguments:
  -h, --help            show this help message and exit
  -topics TOPICS [TOPICS ...]
                        Comma separated list of topics to reassign replicas.
                        Use ALL|all to rebalance all topics
  -execute              whether or not to execute the reassignment plan
  -verify               Verify status of reassignment operation
  -force                Force rebalance of all partitions in a topic, even if already balanaced.
  -throttle             Upper bound on bandwidth used to move replicas from machine to machine.
  -username USERNAME    Username for current user. Required for computing storage details.
  -password PASSWORD    Password for current user. Required for computing storage details.
```

Without "--execute" this tool only scans the current assignment generates the replica reassignment file.

## Example Invocation:

#### Generate reassignment plan for all topics on cluster:

```sudo python rebalance_rackaware.py -topics ALL -username $USERNAME -password $PASSWORD```

The plan will be saved at /var/log/rebalancePlan.json

#### Execute reassignment:

```sudo python rebalance_rackaware.py --execute```

This will execute the plan saved in the above location.

#### Verify progress of reassignment:

```sudo python rebalance_rackaware.py --verify```


## Debugging
Debug logs can be found /var/log/rebalance_log.log.
The log file includes detailed information about the steps taken by the tool and can be used for troubleshooting.