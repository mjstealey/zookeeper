# Apache ZooKeeper in Docker

This work has been inspired by:

- ZooKeeper: [31z4/zookeeper-docker](https://github.com/31z4/zookeeper-docker)
- Oracle Java 8: [binarybabel/docker-jdk](https://github.com/binarybabel/docker-jdk/blob/master/src/centos.Dockerfile)
- CentOS 7 base image: [krallin/tini-images](https://github.com/krallin/tini-images)

### What is ZooKeeper?

ZooKeeper is a centralized service for maintaining configuration information, naming, providing distributed synchronization, and providing group services. All of these kinds of services are used in some form or another by distributed applications. Each time they are implemented there is a lot of work that goes into fixing the bugs and race conditions that are inevitable. Because of the difficulty of implementing these kinds of services, applications initially usually skimp on them ,which make them brittle in the presence of change and difficult to manage. Even when done correctly, different implementations of these services lead to management complexity when the applications are deployed.

See [official documentation](http://zookeeper.apache.org) for more information.

## How to use this image

### Build locally

```
$ docker build -t renci/zookeeper:3.4.11 ./3.4.11/
  ...
$ docker images
REPOSITORY            TAG                 IMAGE ID            CREATED              SIZE
renci/zookeeper       3.4.11              02d4eb4223f6        About a minute ago   691MB
...
```

Example `docker-compose.yml` file included that builds from local repository and deploys 3 node cluster.

```
$ docker-compose build
  ...
$ docker-compose up -d
  ...
$ docker-compose ps
Name              Command               State                     Ports
------------------------------------------------------------------------------------------
zoo1   /usr/local/bin/tini -- /do ...   Up      0.0.0.0:2181->2181/tcp, 2888/tcp, 3888/tcp
zoo2   /usr/local/bin/tini -- /do ...   Up      0.0.0.0:2182->2181/tcp, 2888/tcp, 3888/tcp
zoo3   /usr/local/bin/tini -- /do ...   Up      0.0.0.0:2183->2181/tcp, 2888/tcp, 3888/tcp
```

### From Docker Hub

Automated builds are generated at: [https://hub.docker.com/u/renci](https://hub.docker.com/u/renci/dashboard/) and can be pulled as follows.

```
$ docker pull renci/zookeeper:3.4.11
```

### Start a Zookeeper server instance

	$ docker run --name some-zookeeper --restart always -d renci/zookeeper:3.4.11

This image includes `EXPOSE 2181 2888 3888` (the zookeeper client port, follower port, election port respectively), so standard container linking will make it automatically available to the linked containers. Since the Zookeeper "fails fast" it's better to always restart it.

### Connect to Zookeeper from an application in another Docker container

	$ docker run --name some-app --link some-zookeeper:zookeeper -d application-that-uses-zookeeper

### Connect to Zookeeper from the Zookeeper command line client

	$ docker run -it --rm --link some-zookeeper:zookeeper renci/zookeeper:3.4.11 zkCli.sh -server zookeeper

### ... via [`docker stack deploy`](https://docs.docker.com/engine/reference/commandline/stack_deploy/) or [`docker-compose`](https://github.com/docker/compose)

Example `stack.yml` for `renci/zookeeper:3.4.11`:

```yaml
version: '3.1'

services:
    zoo1:
        image: renci/zookeeper:3.4.11
        restart: always
        hostname: zoo1
        ports:
            - 2181:2181
        environment:
            ZOO_MY_ID: 1
            ZOO_SERVERS: server.1=0.0.0.0:2888:3888 server.2=zoo2:2888:3888 server.3=zoo3:2888:3888

    zoo2:
        image: renci/zookeeper:3.4.11
        restart: always
        hostname: zoo2
        ports:
            - 2182:2181
        environment:
            ZOO_MY_ID: 2
            ZOO_SERVERS: server.1=zoo1:2888:3888 server.2=0.0.0.0:2888:3888 server.3=zoo3:2888:3888

    zoo3:
        image: renci/zookeeper:3.4.11
        restart: always
        hostname: zoo3
        ports:
            - 2183:2181
        environment:
            ZOO_MY_ID: 3
            ZOO_SERVERS: server.1=zoo1:2888:3888 server.2=zoo2:2888:3888 server.3=0.0.0.0:2888:3888
```

This will start Zookeeper in [replicated mode](http://zookeeper.apache.org/doc/current/zookeeperStarted.html#sc_RunningReplicatedZooKeeper). Run `docker stack deploy -c stack.yml zookeeper` (or `docker-compose -f stack.yml up`) and wait for it to initialize completely. Ports `2181-2183` will be exposed.

> Please be aware that setting up multiple servers on a single machine will not create any redundancy. If something were to happen which caused the machine to die, all of the zookeeper servers would be offline. Full redundancy requires that each server have its own machine. It must be a completely separate physical server. Multiple virtual machines on the same physical host are still vulnerable to the complete failure of that host.

Consider using [Docker Swarm](https://www.docker.com/products/docker-swarm) when running Zookeeper in replicated mode.

## Configuration

Zookeeper configuration is located in `/conf`. One way to change it is mounting your config file as a volume:

	$ docker run --name some-zookeeper --restart always -d -v $(pwd)/zoo.cfg:/conf/zoo.cfg renci/zookeeper

## Environment variables

ZooKeeper recommended defaults are used if `zoo.cfg` file is not provided. They can be overridden using the following environment variables.

    $ docker run -e "ZOO_INIT_LIMIT=10" --name some-zookeeper --restart always -d renci/zookeeper

### `ZOO_TICK_TIME`

Defaults to `2000`. ZooKeeper's `tickTime`

> The length of a single tick, which is the basic time unit used by ZooKeeper, as measured in milliseconds. It is used to regulate heartbeats, and timeouts. For example, the minimum session timeout will be two ticks

### `ZOO_INIT_LIMIT`

Defaults to `5`. ZooKeeper's `initLimit`

> Amount of time, in ticks (see tickTime), to allow followers to connect and sync to a leader. Increased this value as needed, if the amount of data managed by ZooKeeper is large.

### `ZOO_SYNC_LIMIT`

Defaults to `2`. ZooKeeper's `syncLimit`

> Amount of time, in ticks (see tickTime), to allow followers to sync with ZooKeeper. If followers fall too far behind a leader, they will be dropped.

### `ZOO_MAX_CLIENT_CNXNS`

Defaults to `60`. ZooKeeper's `maxClientCnxns`

> Limits the number of concurrent connections (at the socket level) that a single client, identified by IP address, may make to a single member of the ZooKeeper ensemble.

### `ZOO_STANDALONE_ENABLED`

Defaults to `false`. Zookeeper's [`standaloneEnabled`](http://zookeeper.apache.org/doc/trunk/zookeeperReconfig.html#sc_reconfig_standaloneEnabled)

> Prior to 3.5.0, one could run ZooKeeper in Standalone mode or in a Distributed mode. These are separate implementation stacks, and switching between them during run time is not possible. By default (for backward compatibility) standaloneEnabled is set to true. The consequence of using this default is that if started with a single server the ensemble will not be allowed to grow, and if started with more than one server it will not be allowed to shrink to contain fewer than two participants.

## Replicated mode

Environment variables below are mandatory if you want to run Zookeeper in replicated mode.

### `ZOO_MY_ID`

The id must be unique within the ensemble and should have a value between 1 and 255. Do note that this variable will not have any effect if you start the container with a `/data` directory that already contains the `myid` file.

### `ZOO_SERVERS`

This variable allows you to specify a list of machines of the Zookeeper ensemble. Each entry has the form of `server.id=host:port:port`. Entries are separated with space. Do note that this variable will not have any effect if you start the container with a `/conf` directory that already contains the `zoo.cfg` file.

In 3.5, the syntax of this has changed. Servers should be specified as such: `server.id=<address1>:<port1>:<port2>[:role];[<client port address>:]<client port>` [Zookeeper Dynamic Reconfiguration](http://zookeeper.apache.org/doc/trunk/zookeeperReconfig.html)

## Where to store data

This image is configured with volumes at `/data` and `/datalog` to hold the Zookeeper in-memory database snapshots and the transaction log of updates to the database, respectively.

> Be careful where you put the transaction log. A dedicated transaction log device is key to consistent good performance. Putting the log on a busy device will adversely affect performance.

## License

View [license information](https://github.com/apache/zookeeper/blob/release-3.4.11/LICENSE.txt) for the software contained in this image.