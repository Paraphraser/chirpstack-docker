# ChirpStack + IOTstack example

This is a fork of the chirpstack/chirpstack-docker repository with modifications intended to permit ChirpStack to run alongside IOTstack.

It should be treated as a work in progress.

## References:

* [Upstream repository](chirpstack/chirpstack-docker](https://github.com/chirpstack/chirpstack-docker/tree/master)
* Original [README.md](https://github.com/chirpstack/chirpstack-docker/blob/master/README.md)

## Directory layout

```
~
├── chirpstack
│   ├── docker-compose.yml
│   ├── configuration
│   │   ├── chirpstack
│   │   ├── chirpstack-gateway-bridge
│   │   └── postgresql
│   │       └── initdb
│   └── volumes
│       ├── postgres
│       │   ├── data
│       │   └── db_backup
│       └── redis
│           ├── data
│           └── db_backup
└── IOTstack
    ├── docker-compose.yml
    │   ├── backups
    │   └── influxdb
    │       └── db
    ├── services
    │   └── nodered
    └── volumes
        ├── grafana
        │   ├── data
        ├── influxdb
        │   └── data
        ├── mosquitto
        │   ├── config
        │   ├── data
        │   ├── log
        │   └── pwfile
        └── nodered
            ├── data
            └── ssh
```

## Significant changes

The significant changes to the original ChirpStack `docker-compose.yml` include:

1. Adding a `networks:` section which declares two networks:

	```
	chirpstack_default
	chirpstack_dbprivate
	```
	
	* Database containers (`postgres` and `redis`) attach to `chirpstack_dbprivate`, thereby keeping inter-container database communications private.

	* The `chirpstack` container is a client of the database containers so it attaches to both networks.

	* The remaining containers attach to `chirpstack_default` by default.

2. Adding a `container_name:` clause to each service definition. This keeps container names succinct in displays such as `docker ps`.

3. Containers with a dependency on `mosquitto` have that dependency removed in favour of an `extra_hosts:` clause linking the hostname `mosquitto` with the `host-gateway` keyword. In practice, means "the `mosquitto` instance running via IOTstack". ChirpStack containers continue to reference `mosquitto:1883` and are unaware that Mosquitto is the IOTstack implementation.

4. The database containers (`postgres` and `redis`) adopt IOTstack conventions with respect to persistent storage (ie each container has a dedicated sub-directory in `./volumes`). This does away with the need for named volume mounts. Also added are paths to holding areas for "live backup" operations.

5. Removes the `mosquitto` service definition. This avoids the two spurious anonymous volume mounts that are created every time the Chirpstack `mosquitto` instance starts. Those are not re-used, are never removed, and serve only to waste disk space. The IOTstack service definition is compatible in that it defaults to allowing anonymous connections but is also structured for easy support of authenticated MQTT connections if those are thought desirable. The IOTstack service definition also includes a health-check.

## Installation

### IOTstack

* **Option 1**

	Use [PiBuilder](https://github.com/Paraphraser/PiBuilder). This installs [IOTstack](https://github.com/SensorsIot/IOTstack), [IOTstackBackup](https://github.com/Paraphraser/IOTstackBackup) and [IOTstackAliases](https://github.com/Paraphraser/IOTstackAliases), along with `docker`, `docker compose` and other tools that are useful for stack operations.
	
* **Option 2**

	Clone [IOTstack](https://github.com/SensorsIot/IOTstack) and run its installer script:
	
	``` console
	$ cd
	$ git clone https://github.com/SensorsIot/IOTstack.git IOTstack
	$ ./IOTstack/install.sh
	```
	
	This results in a "bare bones" installation. You get [IOTstack](https://github.com/SensorsIot/IOTstack), `docker` and `docker compose` but nothing else. In particular, you are on your own for backup and restore solutions ([IOTstackBackup](https://github.com/Paraphraser/IOTstackBackup) still recommended).
	
### ChirpStack

Clone this repo:

``` console
$ cd
$ git clone https://github.com/Paraphraser/chirpstack-docker.git chirpstack
```

Note:

* this differs from the official ChirpStack [getting started](https://www.chirpstack.io/docs/getting-started/docker.html) documentation which does not specify a folder name on the `git clone` command. The result is a folder named `chirpstack-docker` which then becomes the Docker "project name" and, in turn, contributes to excessive wordiness in Docker displays.
* If you make a mistake and omit the `chirpstack` folder name, simply rename the folder:

	``` console
	$ cd
	$ mv chirpstack-docker chirpstack
	```

## Setup

### IOTstack

1. Run the IOTstack menu:

	``` console
	$ cd ~/IOTstack
	$ ./menu.sh
	```
	
2. From the menu, choose "Build Stack" and then select the containers you want. A good starting point is:

	* `grafana` (optional)
	* `influxdb` (optional - do **not** choose `influxdb2`) 
	* `mosquitto` (mandatory - required for ChirpStack)
	* `nodered` (optional)

	If you choose `nodered` press the right arrow and select the default set of add-on nodes. This is a required step if you choose `nodered`.
	
	Note:
	
	* If the menu misbehaves, try enlarging your terminal window. If it keeps misbehaving it is likely the result of your terminal settings. For some reason, misbehaviour seems more pronounced on Ubuntu.
	* A replacement menu system is being worked on.
	
3. After you exit the menu, the `docker-compose.yml` will exist and you can start the stack:

	``` console
	$ docker compose up -d
	```

### ChirpStack

Start the stack:

``` console
$ cd ~/chirpstack
$ docker compose up -d
```

## Expected result:

``` console
$ docker ps -a --format "table {{.Names}}\t{{.RunningFor}}\t{{.Status}}\t{{.Size}}"
NAMES                     CREATED          STATUS                    SIZE
rest-api                  33 seconds ago   Up 31 seconds             4.1kB (virtual 32.6MB)
chirpstack                33 seconds ago   Up 31 seconds             12.3kB (virtual 62.3MB)
postgres                  33 seconds ago   Up 32 seconds             24.6kB (virtual 280MB)
redis                     33 seconds ago   Up 32 seconds             8.19kB (virtual 43.4MB)
chirpstack-basicstation   33 seconds ago   Up 32 seconds             12.3kB (virtual 25.4MB)
chirpstack-gateway        33 seconds ago   Up 32 seconds             12.3kB (virtual 25.4MB)
nodered                   39 seconds ago   Up 38 seconds (healthy)   12.3kB (virtual 760MB)
influxdb                  39 seconds ago   Up 38 seconds (healthy)   16.4kB (virtual 314MB)
mosquitto                 39 seconds ago   Up 38 seconds (healthy)   12.3kB (virtual 17.4MB)
grafana                   39 seconds ago   Up 38 seconds (healthy)   8.19kB (virtual 783MB)
```	
