# Spec Session 1


## Objectives
### A. Understand a flare from docker
1. Know where to find:
	* The env vars used
	* The list of the containers on the host
	* The details about the agent container
2. Know from the flare if the agent is running OK (Health)

### B. Container Architecture
1. Know the high level concept of what a container is and how it exists in a host (permissions, image, etc). Simply put, why do people use it and how is it different from the traditional setup of a VM? 
2. Know the differences between the container installation of the agent versus host installation in terms of command lines to run
3. Have the agent installed on a docker environment getting check metrics (docker at least) and know how to get the status.

---

## Why Do People Use Containers

- Lightweight
	- only has the bare minimum to run the application
- Portability
	- easy to install, easy to delete without leaving any straggler files
- Security
	- a container only has access to files you give it access to. This applies also applies to ports.
- Configuration
	- can limit cpu and memory resources in a container level, making allocations uniform across the board
 - Consistency
	- You can be sure that the app deployed is the final version of the product that you want to run.


## Container Architecture

* Containers: 
	- `docker ps`
	- `docker ps -a`
* Images:
	- `docker images`
* Networks:
	- `docker network ls`
---
*  Logs:
	- `docker logs <container id>` **can get logs from stopped containers
* Container Metadata
	-  `docker inspect`

## Networking

Types of Docker Networks
	* bridge - every container has their own network namespace
	* host - container uses the host network namespace
	* none - none, just none
	
<img src="https://user-images.githubusercontent.com/30991348/87386124-1d6aad80-c5e3-11ea-927b-6e37a17afd3e.png" width="500" height="332" alt="_DSC4652"></a>


## Installation Command
- https://github.com/JM-Lazaro/KB/wiki/Docker-Agent-Installation-Command
	- where to find the configurable env vars
	- how the agent uses the docker socket:
		- uses Go library but uses something like this:
		* `curl --unix-socket /var/run/docker.sock http://localhost/events`
		* `curl --unix-socket /var/run/docker.sock http://localhost/containers/json`
	- how to mount a config file for auto-discovery

Exercise: Install an agent

```
DOCKER_CONTENT_TRUST=1 \
docker run -d \
--name dd-agent1 \
-v /var/run/docker.sock:/var/run/docker.sock:ro \
-v /opt/datadog-agent/run:/opt/datadog-agent/run:rw \
-v /proc/:/host/proc/:ro \
-v /sys/fs/cgroup/:/host/sys/fs/cgroup:ro \
-e DD_API_KEY=<API_KEY> \
-e DD_LOGS_ENABLED=true \
-e DD_LOGS_CONFIG_CONTAINER_COLLECT_ALL=true \
datadog/agent:7
```


## Ability to understand a flare from container agent
Sample Flare: https://datadog.zendesk.com/agent/tickets/316095

How do you know if the flare is from a container?
	- docker_inspect.yaml ***
	- datadog.yaml
	- diagnose.yaml
	- docker_ps.yaml


	

Exercise:
1. create an nginx container in default
	* `docker run -d --name defaultginx nginx:latest`
	* `netstat -plnt | grep 80`
2. create an nginx container in Host
	* `docker run -d --name hostginx --network host nginx:latest`
	* `netstat -plnt | grep 80`
3. create an nginx container in None
	* `docker inspect noneginx`
4. create a test bridge network
	* `docker create network test-network`
	* `docker run -d --name testginx --network test-network nginx:latest`
5. connect the test container to default bridge
	* `docker network connect bridge testginx`


## Dogstatsd and APM
For Dogstatsd, you'll need to add:
```
-e DD_DOGSTATSD_NON_LOCAL_TRAFFIC=true
-p 8125:8125/udp
```

Test:
```
echo -n "custom_metric_host_docker0:60|g|#shell" | nc -v -u -w 1 <IP Address> 8125
```

For APM:
```
-p 8126:8126/tcp
```

Test:

	* run as `./script_name.sh <ip address>`

```
#!/bin/bash

# Create IDs.
TRACE_ID=($RANDOM % 1000000)
SPAN_ID=($RANDOM % 1000000)

# Start a timer.
# For the Alpine image, in order to get the time in nanoseconds,
# you may need to install coreutils ("apk add coreutils").
START=$(date +%s%N)

# Do things...
sleep 2

# Stop the timer.
DURATION=$(($(date +%s%N) - $START))

# Send the traces.
curl -X PUT -H "Content-type: application/json" \
  -d "[[{
    \"trace_id\": $TRACE_ID,
    \"span_id\": $SPAN_ID,
    \"name\": \"span_name\",
    \"resource\": \"/home\",
    \"service\": \"service_name\",
    \"type\": \"web\",
    \"start\": $START,
    \"duration\": $DURATION
}]]" \
  http://$1:8126/v0.3/traces
```

