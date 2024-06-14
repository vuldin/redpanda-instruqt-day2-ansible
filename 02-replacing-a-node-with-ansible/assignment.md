---
slug: replacing-a-node-with-ansible
id: 4hzaohwabyby
type: challenge
title: Replacing a node with Ansible
teaser: A short description of the challenge.
notes:
- type: text
  contents: Disaster! A node will fail... your task is to replace it. Onwards and
    upwards!
tabs:
- title: Shell
  type: terminal
  hostname: server
difficulty: basic
timelimit: 600
---
# Validate Cluster Health

```bash,run
export REDPANDA_BROKERS="node-a:9092,node-b:9092,node-c:9092"
rpk cluster health
```

Output:

```bash,nocopy
CLUSTER HEALTH OVERVIEW
=======================
Healthy:                          true
Unhealthy reasons:                []
Controller ID:                    0
All nodes:                        [0 1 2]
Nodes down:                       []
Leaderless partitions (0):        []
Under-replicated partitions (0):  []
```

# Kill Redpanda on Node-A

```bash,run
ssh -o StrictHostKeychecking=no node-a killall redpanda
```

# Recheck Health

```bash,run
rpk cluster health
```

Output:

```bash,nocopy
unable to request cluster health: Get "http://node-a:9644/v1/cluster/health_overview": dial tcp 10.192.0.20:9644: connect: connection refused
```
Notice that rpk can no longer connect to node-a... this makes sense, since we just killed the broker on that node!

Let's change our environment variable so that node-b is first in the list:

```bash,run
export REDPANDA_BROKERS="node-b:9092,node-c,node-a:9092"
rpk cluster health
```

Output:

```bash,nocopy

CLUSTER HEALTH OVERVIEW
=======================
Healthy:                          false
Unhealthy reasons:                [nodes_down under_replicated_partitions]
Controller ID:                    1
All nodes:                        [0 1 2]
Nodes down:                       [0]
Leaderless partitions (0):        []
Under-replicated partitions (7):  [kafka/log1/0 kafka/log2/1 kafka/log3/0 kafka/log3/2 kafka/log4/0 kafka/log5/1 redpanda/controller/0]
```

# Add a New Node

Let's add node-d to our `hosts.ini` file, while also marking the original nodes with `skip_node=true`

```bash,run
cd deployment-automation
IP_A=$(nslookup node-a | grep Address | tail -1 | cut -f2 -d' ')
IP_B=$(nslookup node-b | grep Address | tail -1 | cut -f2 -d' ')
IP_C=$(nslookup node-c | grep Address | tail -1 | cut -f2 -d' ')
IP_D=$(nslookup node-d | grep Address | tail -1 | cut -f2 -d' ')

cat << EOF > hosts.ini
[redpanda]
node-a ansible_user=root ansible_become=True private_ip=${IP_A} id=0 skip_node=true
node-b ansible_user=root ansible_become=True private_ip=${IP_B} id=1 skip_node=true
node-c ansible_user=root ansible_become=True private_ip=${IP_C} id=2 skip_node=true
node-d ansible_user=root ansible_become=True private_ip=${IP_D} id=3
EOF

cat hosts.ini
```

Output:

```bash,nocopy
[redpanda]
node-a ansible_user=root ansible_become=True private_ip=10.192.0.20 id=0 skip_node=true
node-b ansible_user=root ansible_become=True private_ip=10.192.0.17 id=1 skip_node=true
node-c ansible_user=root ansible_become=True private_ip=10.192.0.93 id=2 skip_node=true
node-d ansible_user=root ansible_become=True private_ip=10.192.0.19 id=3
```

# Run the Playbook to Install the New Node

```bash,run
export DEPLOYMENT_PREFIX=instruqt
export ANSIBLE_COLLECTIONS_PATH=${PWD}/artifacts/collections
export ANSIBLE_ROLES_PATH=${PWD}/artifacts/roles
export ANSIBLE_INVENTORY=${PWD}/artifacts/hosts_gcp_$DEPLOYMENT_PREFIX.ini

ansible-playbook --private-key ~/.ssh/id_rsa -v ansible/provision-cluster.yml -i hosts.ini
```

# Decommission the Failed Node

First, identify the broker ID of the failed broker:

```bash,run
rpk cluster health
```

Output:

```bash,nocopy

CLUSTER HEALTH OVERVIEW
=======================
Healthy:                          false
Unhealthy reasons:                [nodes_down under_replicated_partitions]
Controller ID:                    1
All nodes:                        [0 1 2]
Nodes down:                       [0]
Leaderless partitions (0):        []
Under-replicated partitions (7):  [kafka/log1/0 kafka/log2/1 kafka/log3/0 kafka/log3/2 kafka/log4/0 kafka/log5/1 redpanda/controller/0]
```

Notice the line `Nodes down: [0]` - this means that broker ID 0 is currently down.

At this point, we want to force the decommission of broker 0:

```bash,run
rpk redpanda admin brokers decommission 0 --force
```

Output:
```bash,nocopy
Success, broker 0 decommission started.  Use `rpk redpanda admin brokers decommission-status 0` to monitor data movement.
```

As suggested by RPK, we can monitor the decommissioning process with the following command:
```bash,run
rpk redpanda admin brokers decommission-status 0
```

Output:
```bash,nocopy
Node 0 is decommissioned successfully.
```

Once the output matches the above (decommissioned successfully), we have successfully completed the steps required.

## Validate the Cluster Health
```bash,run
rpk cluster health
```

Output:

```bash,nocopy
CLUSTER HEALTH OVERVIEW
=======================
Healthy:                          true
Unhealthy reasons:                []
Controller ID:                    1
All nodes:                        [1 2 3]
Nodes down:                       []
Leaderless partitions (0):        []
Under-replicated partitions (0):  []
```

Notice that the cluster is now healthy, and broker ID 0 is not mentioned anywhere in the output.

# Summary

We created killed a node and added one to replace it. We decommissioned the failed node and watched while the cluster
healed itself. Success!