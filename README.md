# Splunk Search Head Clutser Testing on Docker Swarm
This was developed with the intention of replacing the search heads in an existing Indexer Cluster all running on bare metal servers.

## Topolgy:

Splunk Deployer
Splunk Indexer
Splunk Search Heads (scaled min 3)
Traefik (loadbalancer)

## Environment Setup:

At least 3 PhotonOS VMs with docker in swarm mode.
- `node01`  docker swarm leader
- `node02`  docker swarm member
- `node03`  docker swarm member

### Docker Swarm Setup:

#### 1. Setup iptables
````
iptables -A INPUT -p tcp --dport 2377 -j ACCEPT # only on swarm leader
iptables -A INPUT -p tcp --dport 2376 -j ACCEPT
iptables -A INPUT -p tcp --dport 7946 -j ACCEPT
iptables -A INPUT -p udp --dport 7946 -j ACCEPT
iptables -A INPUT -p udp --dport 4789 -j ACCEPT
````
*NOTE: this is does not persistent after reboot*

[Reference] https://www.digitalocean.com/community/tutorials/how-to-configure-the-linux-firewall-for-docker-swarm-on-centos-7

#### 2. Setup Docker Docker Swarm
Run following command: 
````
docker swarm init
````
Run the generated join command on other nodes.

[Reference] https://docs.docker.com/engine/reference/commandline/swarm_init/

Give a label to the `node01` docker swarm instance:
````
export NODE_ID=$(docker info -f '{{.Swarm.NodeID}}')
docker node update --label-add traefik-public.traefik-public-certificates=true $NODE_ID 
````
## Running the project:
*[IMPORTANT] Validate files have the appropriate permissions for global read*

#### 1. Get code base on all nodes.
````
tdnf install git -y
mkdir -p /opt/docker/
cd /opt/docker/
git clone https://github.com/rskntroot/splunk
````
#### 2. Run the stack from `node01`
````
cd /opt/docker/splunk/
docker stack deploy splunk -c docker-compose.yml
````
On your host update your `hosts` config:
````
X.X.X.X            node01 traefik.rskdev.io deployer.rskdev.io search.rskdev.io index.rskdev.io
X.X.X.X            node02
X.X.X.X            node03
````
Check services in traefik:
````
http://traefik.rskdev.io:8080
````
Connect to splunk deployer:
````
http://deployer.rskdev.io
````

## Issue
The search heads fail to come online!

This seems to be due to splunk's ansible setup requiring communication between nodes before the swarm publishes the nodes in docker's DNS.

You will see the docker service containers continue to become `unhealthy` then get torn down and rebuilt by docker swarm.

## Work Around
### 1. Remove Existing Stack
````
docker stack rm splunk
watch docker node ps $(docker node ls -q)
````
*Wait for services to fully close*

### 2. Start Search nodes without `SPLUNK_ROLE` assignments
Comment out the following from `search01` `search02` `search03` docker-compose.yml:
````
#      - SPLUNK_SEARCH_HEAD_URL=search02,search03
#      - SPLUNK_SEARCH_HEAD_CAPTAIN_URL=search01
#      - SPLUNK_ROLE=splunk_search_head
```` 
### 3. Start the Stack
````
docker stack deploy splunk -c docker-compose.yml
watch docker node ps $(docker node ls -q)
````
### Manually Configure Search Head Cluster:
*deployer has already been configured*

For each [search] container across nodes:
`$SERVICE_ID` is the actual docker container name or hash from `docker ps`
`$CONF_PASS` is the admin password for the splunkd instance
`$CONF_KEY` is `splunk.shc.pass4SymmKey` in `/opt/docker/splunk/default.yml`
`$CONF_LABEL` is `splunk.shc.label` in `/opt/docker/splunk/default.yml`
````
docker exec -it $SERVICE_ID /bin/bash
````
````
sudo -i
ln -s /opt/splunk/bin/splunk /usr/local/sbin/.
splunk init shcluster-config -auth admin:$CONF_PASS -mgmt_uri https://$(hostname):8089 \
-replication_port 34567 -replication_factor 3 -conf_deploy_fetch_url https://deployer:8089 \
-secret $CONF_KEY -shcluster_label $CONF_LABEL
splunk restart
````
One of the nodes needs to be bootstrapped as the shc captain: 
````
splunk bootstrap shcluster-captain -auth admin:$CONF_PASS \
-servers_list "https://search01:8089,https://search02:8089,https://search03:8089"
````
Watch the shc until all nodes show up:
NOTE: Some search containers will randomly refuse to join the captain and may require one or `splunk restart` commands to get them going.
````
watch splunk show shcluster-status --verbose
````
Check the status of the shc via `webgui`
````
http://search.rskdev.io/en-US/manager/system/search_head_clustering
````

In testing I have found:
- search functionality, `working` as expected.
- app deployment, `working` as expected.
- artifact replication, `working` as expected.
- container persistent, `fails` as expected (and takes out the shc as well).
- deployed dashboards may `fail` to run, but the searchs work.

## Additional:

I'm at wits-end on this one and was wondering if anyone wants to give me some pointers on how to create an ansible playbook for this case 🤷🏻‍♂️



