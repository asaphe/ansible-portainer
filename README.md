# Ansible Portainer for Docker Swarm
This role is designed to deploy Portainer version 1.19 onto a Docker Swarm Cluster as a stack.
The stack has two type of containers, a *Manager* container and an *Agent* container.
The manager uses the agent to perform API calls to the Docker daemon where the agent is running.

* OS: Ubuntu 16.04+

### Requirements
- Docker Swarm
- Docker Engine API
- Persistent Data directory

>Persistent Data: A Docker Volume will be defined if `portainer.volume_path` is not defined  


**Our specific implementation relies on OneLogin SSO and on AWS EFS (Terraform)**

### Role Variables
A combination of control variables and other variables is designed to lend flexibility to the role.

**Role Control**  

|Type  |Name              |Default Value |Description                                           |
|:----:|:----------------:|:------------:|:----------------------------------------------------:|
|bool  |skip_create_admin |False         |when `True` skip the creation of an admin user and setting the password  |
|bool  |skip_endpoints    |False         |when `True` skip the configuration of additional endpoints |
|bool  |skip_deploy       |False         |when `True` skip the deployment of the Docker Stack |
|bool  |skip_configure    |False         |when `True` skip the configuration of Portainer via the role|
|bool  |remove_stack      |False         |when `True` will remove an existing Docker Stack (stack name is the same as the deployed stack |
|dict  |registry.enabled  |False         |when `Defined` will configure the Docker registries as defined in the role  |
 
**Mandatory**  

|Type  |Name             |Default Value |Description                                           |
|:----:|:---------------:|:------------:|:--------------------------------------------------- :|
|dict  |portainer        |N/A           | A dictionary containing container settings used in the role and in the Docker-Stack template |
|dict  |portainer_config |N/A           | A dictionary containing Portainer settings used to template the JSON |

**Optional**  

|Type  |Name             |Default Value |Description                                           |
|:----:|:---------------:|:------------:|:--------------------------------------------------- :|
|dict  |portainer_sso    |N/A           | A dictionary containing settings required for enabling SSO via LDAP |
|dict  |registry         |N/A           | A dictionary containing settings for a registry to be defined in Portainer |
|list  |endpoints        |N/A           | A list of **dictionaries** which is later flattened and used to define additional endpoints via the API|

>**Endpoints list example**  
```yaml
endpoints:
  - {name: local, url: "unix:///var/run/docker.sock"}
  - {name: remote1, url: "tcp://1.2.3.4:2375"}
  - {name: remote2, url: "tcp://5.6.7.8:2375"}
```

>**Registry dictionary example**  
```yaml
registry:
  registry_name: nexus-oss
  registry_url: 1.2.3.4
  registry_port: 5001
  registry_auth: false
  registry_username: username
  registry_password: password
  enabled: false
```

#### Tags
```
docker.swarm.portainer.configure
docker.swarm.portainer.configure.admin
docker.swarm.portainer.configure.settings.endpoints
docker.swarm.portainer.configure.settings
docker.swarm.portainer.configure.registry
docker.swarm.portainer.remove
docker.swarm.portainer.deploy
```

### Docker Engine API
The Engine API is an HTTP API served by Docker Engine. It is the API the Docker client uses to communicate with the Engine, so everything the Docker client can do can be done with the API.
Most of the client's commands map directly to API endpoints (e.g. docker ps is GET /containers/json). The notable exception is running containers, which consists of several API calls.

* The daemon listens on unix:///var/run/docker.sock but you can Bind Docker to another host/port or a Unix socket.
* The API tends to be REST. However, for some complex commands, like attach or pull, the HTTP connection is hijacked to transport stdout, stdin and stderr.
* A Content-Length header should be present in POST requests to endpoints that expect a body.
* To lock to a specific version of the API, you prefix the URL with the version of the API to use. For example, /v1.18/info. If no version is included in the URL, the maximum supported API version is used.
* If the API version specified in the URL is not supported by the daemon, a HTTP 400 Bad Request error message is returned.

API documentation can be found [here](https://docs.docker.com/engine/api/latest/#38-swarm)

#### Configure the Docker daemon
There are two ways to configure the Docker daemon:
- Use a JSON configuration file. This is the preferred option, since it keeps all configurations in a single place.
- Use flags when starting dockerd.

>You can use both of these options together as long as you don’t specify the same option both as a flag and in the JSON file. If that happens, the Docker daemon won’t start and prints an error message.

On Ubuntu 16.04 for example, we've added to Docker's service file:
```raw
ExecStart=/usr/bin/dockerd -H fd:// -H tcp://0.0.0.0:2376
```
>File Location: `/lib/systemd/system/docker.service`
>By default the `ExecStart` directive doesn't enable the `tcp` endpoint

The preferred way would be the JSON file located at `/etc/docker/daemon.json`
Example of a valid JSON:
```json
{
  "debug": true,
  "tls": true,
  "tlscert": "/var/docker/server.pem",
  "tlskey": "/var/docker/serverkey.pem",
  "hosts": ["tcp://192.168.59.3:2376"]
}
```

The same thing can be achieved using the CLI:
```shell
dockerd --debug \
  --tls=true \
  --tlscert=/var/docker/server.pem \
  --tlskey=/var/docker/serverkey.pem \
  --host tcp://192.168.59.3:2376

```

Useful reading about [troubleshooting](https://docs.docker.com/config/daemon/#troubleshoot-conflicts-between-the-daemonjson-and-startup-scripts) the daemon

### Portainer SSL
By default, Portainer’s web interface and API is exposed over HTTP. This is not secured, it’s recommended to enable SSL in a production environment.  
The following are required when enabling SSL:
- --ssl
- --sslcert
- --sslkey

CLI Example:  
`docker run -d -p 443:9000 --name portainer --restart always -v ~/local-certs:/certs -v portainer_data:/data portainer/portainer --ssl --sslcert /certs/portainer.crt --sslkey /certs/portainer.key`
Generate Certificates Example:  
```shell
$ openssl genrsa -out portainer.key 2048
$ openssl ecparam -genkey -name secp384r1 -out portainer.key
$ openssl req -new -x509 -sha256 -key portainer.key -out portainer.crt -days 3650
```

##### To-Do
This role can be even more flexible, define multiple registries etc,.
