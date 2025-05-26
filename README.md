## Docker Swarm Setup

**Docker Swarm** is Docker's built-in tool for orchestrating containers across a cluster of machines. It manages a group of Docker engines and enables high availability, scaling, and service deployment.

### Cluster Topology

* 1 Swarm Manager Node
* 2 Worker Nodes

---

### Setup Instructions

Run the following Ansible playbooks in order:

#### 1. `playbooks/main.yml`

This playbook provisions and initializes the Docker Swarm cluster.

```yaml
- name: Docker Swarm Cluster Setup
  hosts: all
  become: true
  roles:
    - common
```

* Installs Docker and other necessary packages on all nodes.

```yaml
- name: Initialize Manager Node
  hosts: manager
  become: true
  roles:
    - manager
```

* Initializes the Swarm on the manager node using `docker swarm init`.

```yaml
- name: Join Worker Nodes
  hosts: workers
  become: true
  roles:
    - worker
```

* Retrieves the join token from the manager and runs `docker swarm join` on the worker nodes.

#### 2. `playbooks/stack.yml`

This playbook deploys a service stack on the Swarm manager node.

```yaml
- name: Docker Stack
  hosts: manager
  become: true
  roles:
    - stack
```

* Uses `docker stack deploy` to launch the stack defined in a `docker-compose.yml` file.

### Observations

#### Swarm Manager

```
root@debian:~# docker node ls
ID                            HOSTNAME   STATUS    AVAILABILITY   MANAGER STATUS   ENGINE VERSION
bbbdhyfgfjdgyjc7wlq258v25     debian     Ready     Active                          28.1.1
f15pts5qubirm4fgt209ls8qo *   debian     Ready     Active         Leader           28.1.1
lgypt9ohp72p4e73r8riajy9l     debian     Ready     Active                          28.1.1
```

```
#MANAGER
root@debian:~# docker info
Swarm: active
  NodeID: f15pts5qubirm4fgt209ls8qo
  Is Manager: true
  ClusterID: mkap0cufku05bi2c8av62mjj1
  Managers: 1
  Nodes: 3
  Data Path Port: 4789
  Orchestration:
   Task History Retention Limit: 5
```

```
root@debian:~# docker stack ls
NAME      SERVICES
mystack   1
root@debian:~# docker stack services mystack
ID             NAME            MODE         REPLICAS   IMAGE          PORTS
72trizhs1qxe   mystack_nginx   replicated   4/4        nginx:alpine   *:8080->80/tcp
root@debian:~# docker service ps mystack_nginx
ID             NAME              IMAGE          NODE      DESIRED STATE   CURRENT STATE            ERROR     PORTS
ye7lrni9yo5o   mystack_nginx.1   nginx:alpine   debian    Running         Running 47 seconds ago
s8b877c3gw01   mystack_nginx.2   nginx:alpine   debian    Running         Running 51 seconds ago
sa4qevpk9a8b   mystack_nginx.3   nginx:alpine   debian    Running         Running 47 seconds ago
k0251elc2vy3   mystack_nginx.4   nginx:alpine   debian    Running         Running 52 seconds ago
```

```
# Empty i.e no containers running in manager node

root@debian:~# docker ps
CONTAINER ID   IMAGE     COMMAND   CREATED   STATUS    PORTS     NAMES
```

```
# inspect a specific node

root@debian:~# docker node inspect bbbdhyfgfjdgyjc7wlq258v25 --pretty
ID:			bbbdhyfgfjdgyjc7wlq258v25
Hostname:              	debian
Joined at:             	2025-05-23 16:32:26.051438046 +0000 utc
Status:
 State:			Ready
 Availability:         	Active
 Address:		10.0.0.26
Platform:
 Operating System:	linux
 Architecture:		x86_64
Resources:
 CPUs:			2
 Memory:		3.823GiB
Plugins:
 Log:		awslogs, fluentd, gcplogs, gelf, journald, json-file, local, splunk, syslog
 Network:		bridge, host, ipvlan, macvlan, null, overlay
 Volume:		local
Engine Version:		28.1.1
```

### Swarm Workers

```
root@debian:~# docker node ls
Error response from daemon: This node is not a swarm manager. Worker nodes can't be used to view or modify cluster state. Please run this command on a manager node or promote the current node to a manager.
```

```
#WORKER 1
root@debian:~# docker info
Swarm: active
  NodeID: bbbdhyfgfjdgyjc7wlq258v25
  Is Manager: false
  Node Address: 10.0.0.26
  Manager Addresses:
   10.0.0.25:2377
```

```
#WORKER 2
root@debian:~# docker info
 Swarm: active
  NodeID: lgypt9ohp72p4e73r8riajy9l
  Is Manager: false
  Node Address: 10.0.0.27
  Manager Addresses:
   10.0.0.25:2377
```

```
# Two containers running in each of the worker nodes

root@debian:~# docker ps
CONTAINER ID   IMAGE          COMMAND                  CREATED         STATUS         PORTS     NAMES
4cd219a7867a   nginx:alpine   "/docker-entrypoint.…"   2 minutes ago   Up 2 minutes   80/tcp    mystack_nginx.4.k0251elc2vy3tcbsekofiuv8r
4949acd0139c   nginx:alpine   "/docker-entrypoint.…"   2 minutes ago   Up 2 minutes   80/tcp    mystack_nginx.2.s8b877c3gw0177huhihb7y2xi
```

```
root@debian:~# docker ps
CONTAINER ID   IMAGE          COMMAND                  CREATED         STATUS         PORTS     NAMES
6d92dd95a97b   nginx:alpine   "/docker-entrypoint.…"   2 minutes ago   Up 2 minutes   80/tcp    mystack_nginx.1.ye7lrni9yo5ow1g7ce9s09cur
696deeae8f87   nginx:alpine   "/docker-entrypoint.…"   2 minutes ago   Up 2 minutes   80/tcp    mystack_nginx.3.sa4qevpk9a8b4mpzx2wjrbpvw
```
