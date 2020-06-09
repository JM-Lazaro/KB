
#### Create Your Docker Environment

Follow the KB: https://github.com/DataDog/se-docs/wiki/Quick-Way-to-Install-Docker-Compose-on-Ubuntu-16.04

#### Installation Command

```
DOCKER_CONTENT_TRUST=1 docker run -d --name dd-agent -v /var/run/docker.sock:/var/run/docker.sock:ro -v /proc/:/host/proc/:ro -v /sys/fs/cgroup/:/host/sys/fs/cgroup:ro -e DD_API_KEY={{APIKEY}} datadog/agent:7
```

`DOCKER_CONTENT_TRUST`
* makes sure that the image you are downloading from docker hub is signed by the publisher

`docker run`
* creates and starts the docker container

`-d`
* detach (like nohup), run the container in the background
			
`--name dd-agent`
* sets the container name
			
`-v /var/run/docker.sock:/var/run/docker.sock:ro`
* https://medium.com/better-programming/about-var-run-docker-sock-3bfd276e12fd
* it's the unix socket that the Docker daemon listens to, giving the container access to Docker API	
* docker events, docker logs, docker inspect
			
`-v /proc/:/host/proc/:ro`
* where we get most host metrics e.g. `system.io.*`  are from `/proc/diskstats`
			
`-v /sys/fs/cgroup/:/host/sys/fs/cgroup:ro`
* where we get container utilization metrics any `docker.*
			
`-e DD_API_KEY={{API_KEY}}`
`-e DD_LOG_LEVEL=debug`
* config.go -  [https://github.com/DataDog/datadog-agent/blob/master/pkg/config/config.go](https://github.com/DataDog/datadog-agent/blob/master/pkg/config/config.go) 
* env.go -  [https://github.com/DataDog/datadog-agent/blob/master/pkg/trace/config/env.go](https://github.com/DataDog/datadog-agent/blob/master/pkg/trace/config/env.go) 

`datadog/agent:latest`
* docker image and tag


#### Install the Agent:

Deploy Docker Agent container:

```
DOCKER_CONTENT_TRUST=1 docker run -d \
--name dd-agent \
-v /var/run/docker.sock:/var/run/docker.sock:ro \
-v /proc/:/host/proc/:ro \
-v /sys/fs/cgroup/:/host/sys/fs/cgroup:ro \
-e DD_API_KEY={{APIKEY}} \
datadog/agent:7
```

##### Try these commands:

* `docker ps`
* `docker ps -a` # shows stopped containers
* `docker stop dd-agent`
* `docker start dd-agent`
* `docker images`
* `docker inspect <container id>`
* `docker logs dd-agent`

##### Run agent commands:

* `docker exec -it dd-agent agent status`
* `docker exec -it dd-agent agent configcheck`

##### Access the agent container:

`docker exec -it dd-agent bash`

`agent status`
`agent configcheck`

* check out the directory structure e.g. `/etc/datadog-agent/`

---

#### Basic Networking

Objectives:
	- learn the ways containers can talk to each other
	- learn how the host can talk to containers and vice versa

##### Overview:

##### Four different types of Docker network:
* Bridge - default
* Host
* None
* Custom Bridge

##### Test Container Connections

Install nginx container

* `docker run -d --name mynginx nginx:latest`

Check the Container IP using `docker inspect <container name>`

Ex:
```
            "Networks": {
                "bridge": {
					[...]
                    "Gateway": "172.17.0.1",
                    "IPAddress": "172.17.0.2",
```

** Take note of the agent's and nginx's container IP.

Check the connection from host to containers:

* `curl <mynginx container IP>:80`

Check connection between containers:

* `docker exec -it dd-agent curl <nginx IP>:80`
* `docker exec -it mynginx curl <dd-agent>:8126/v0.4/traces ` <em>## The agent doesn't have a landing page on its ports but you should see something like an "EOF" message when you curl this endpoint.</em>

> TL;DR: 
> 1. Containers and the host can use container IPs to talk to a container.

Customers don't want to use container IPs because they are transient in nature - containers are deleted and recreated all the time, their IP changes each time.

##### Host Ports

Delete the nginx pod:

`docker stop mynginx && docker rm mynginx`

Recreate them with `-p` option. This option maps the container ports to a host port:

* `docker run -d -p 80:80 --name mynginx nginx:latest`

https://docs.docker.com/engine/reference/commandline/run/

Check if the port mapping is successful:

* `docker ps`

```
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                NAMES
dff9e0d18c1d        nginx               "nginx -g 'daemon of…"   34 minutes ago      Up 25 minutes       0.0.0.0:80->80/tcp   nginx
```

Check the connection from the host:

* `curl localhost:80`

or with Host IPs:

* `hostname -I` # one of these IPs is the Docker bridge Gateway IP
* `curl <Host IP>:80`


> TL;DR:
> 1. Containers can map their ports to a host port using the `--publish`  option.
> 2. Containers can direct traffic to the host using the Host IP or Docker Gateway IP.
> 3. Alternatively, they can use `--net=host` but this is not recommended.


#### Integration AutoDiscovery

https://docs.datadoghq.com/agent/autodiscovery/integrations/?tab=docker

There are two(three) ways of implementing autodiscovery with Docker:
* Config Files
* Docker Labels
* Key-Value Store

##### Config Files

Create a local config file. For example:

```
cat << EOF > auto_conf.yaml
ad_identifiers:
  - nginx
init_config:
instances:
  - name: tcp_test
    host: "%%host%%"
    port: "%%port%%"
EOF
```

Mount the config file to the container's config directory:

```
DOCKER_CONTENT_TRUST=1 docker run -d \
--name dd-agent \
-v /var/run/docker.sock:/var/run/docker.sock:ro \
-v /proc/:/host/proc/:ro \
-v /sys/fs/cgroup/:/host/sys/fs/cgroup:ro \
-v <path_to_file>/auto_conf.yaml:/etc/datadog-agent/conf.d/tcp_check.d/auto_conf.yaml \
-e DD_API_KEY={{APIKEY}} \
datadog/agent:7
```
Verify this with `agent status` / `agent configcheck`

> Note: Mounting a file just makes the file accessible to the container. Any changes to the file will be reflected in the container and the host.

##### Docker Labels

Create an nginx container with auto-discovery labels:

```
docker run -d \
--name tcp_nginx \
-l com.datadoghq.ad.check_names='["tcp_check"]' \
-l com.datadoghq.ad.init_configs='[{}]' \
-l com.datadoghq.ad.instances='[{"name": "app_container","host": "%%host%%","port": "%%port%%"}]' \
nginx:latest
```

Verify this with `agent status` / `agent configcheck`

### Dogstatsd

Create a simple custom image that will send a dogstatsd metric to the agent:

```
mkdir DOGSTATSD;

cd DOGSTATSD;

cat << EOF > dogstatsd.bash
#!/bin/bash
while true; do
echo -n "custom_metric:60|g|#shell" | nc -v -u -w 1 \$DD_AGENT_HOST 8125
sleep 1;
done
EOF

cat << EOF > Dockerfile
FROM debian:9
COPY ./dogstatsd.bash /
RUN chmod 777 ./dogstatsd.bash
RUN apt-get update && apt-get install netcat -y
ENTRYPOINT ["/dogstatsd.bash"]
EOF

docker build -t test/image:1.0 .
```

## Sending Metrics to the Agent

Run the custom container with the agent container IP or gatewayIP

`docker run -d --name test_app -e DD_AGENT_HOST={{agentIP or gatewayIP}} test/image:1.0`

> This applies to APM tracers as well. The tracer needs the Agent's IP declared as its agent host along with port 8126 if necessary.

If we have time:

1. Use `--link`
