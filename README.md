# Goals

We want to setup highly redundant cassandra service that:

1. Supports 1+ million of transactions per day.
2. Is highly available and supports software upgrade without downtime.
3. Supports auto-scaling up and down in response to load.

Optionally, extend the infrastructure to support any type of service that uses
binary protocol.

## Considerations for setting up a service

Cassandra already supports decentralized clustering and is fault tolerant which
makes things easier for this experiment. It is easy to setup a cassandra cluster
spanning multiple datacenters. Setting up another service will not be this easy.

## Architecture

The actual number of cassandra nodes will depend heavily on number of concurrent
clients and the type of workload (how many reads per second, how many writes per
second etc) they are doing. In this example, we will setup 5 node cassandra
cluster which will be more than sufficient for 1 million transactions per day.
The number 5 is chosen for ease of supporting fault tolerance. Out of the 5
nodes, 3 nodes will be setup as seed nodes and the other 2 as non-seed nodes.

In addition to these, we shall have 2 HAProxy nodes (master + slave) which will
load balance the incoming connections to the cassandra cluster at TCP level.
This allows the cassandra clients to be completely unaware of the actual node
that services the request and the entire communication is binary. The drawback
is that we won't be able to cache anything along the way for improving
performance using caching servers.

![Cassandra as a service stack](https://raw.githubusercontent.com/chyrag/caas/main/caas.svg "Cassandra as a service stack")

## Software upgrade workflow for upgrading cassandra nodes

Cassandra nodes will run as docker containers[^1] on individual nodes dedicated
for them. Care needs to be taken while setting up the containers for the first
time. The first cassandra node shall be the seed server. The next 2 cassandra
nodes need to use the first node and itself as the seed servers. This is to
enable the nodes to find each other. The remaining cassandra nodes need to use
IP addresses of all the 3 initial nodes as seed servers.

Assume that we have a CI pipeline which builds the latest version of cassandra
upon each commit to the code repository, tests against various unit tests,
sanity tests, integration tests etc and pushes the image to a docker repository.
Once the docker image is available in the docker repository we will use an
ansible playbook to upgrade the nodes serially. Prior to taking down the
cassandra container on a node, we should allow the container to drain the
incoming requests queue or the requests in flight. Cassandra provides a tool
(nodetool) which allows us to drain the requests and prepare for shutdown. Once
the outstanding requests have been processed, the cassandra container can be
brought down. This is a good time to upgrade supporting OS packages on the node
as well. Once the OS packages are upgraded, we pull down the new cassandra
docker image and put it into service. After the cassandra container is started,
we need to run `nodetool upgradesstables` to ensure that the sstables are
rewritten to match the current cassandra version. Before starting to upgrade
another node, we should wait for the upgraded node to join the cluster. This is
done by monitoring the output of nodetool status. The ansible playbook would
upgrade all the nodes in this manner.

## Software upgrade workflow for upgrading HAProxy nodes

HAProxy nodes will be configured in master-slave configuration and shall have a
virtual IP managed using keepalived and VRRP. HAProxy and keepalived will be
installed on the provisioned VM or baremetal instead of using docker
containers[^2].

The slave HAProxy node will be upgraded first using the distribution package
manager (apt-get, yum, dnf) and then the master. There is no requirement to
drain outstanding requests in HAproxy since each request is at TCP level and
will be retried automatically after transmit timeout.

## Scaling up and down

We need to choose specific metrics to monitor to help us decide whether to scale
up or down. The right set of metrics will be the ones that affects the customer
experience and hence affect the business. In case of cassandra, we could measure
the following over a period of time or use tools such as cassandra-stress to
stress the system and determine threshold values for scaling up or down.

1. Read Latency or Write Latency
2. Storage
3. CPU utilization

Ideally, we would like the read/write latency within 10s of milliseconds,
storage within 80% of the configured storage and CPU utilization within 10% (to
accommodate short bursts of requests). This isn't exhaustive list of metrics and
there could be more but these are the initial metrics which will be important.

Note: cassandra can exhibit high read/write latency because of misconfiguration
as well.

Kubernetes provides elastic scaling out of box with simple configuration by
specifying the minimum number of nodes and maximum number of nodes that we wish
to maintain. So, if we are using Kubernetes there is not much to do.

Assuming that we do not have Kubernetes, we would setup a stack for monitoring
our cluster. The choice of monitoring stack will depend largely on what we want
to do with the anomalies. TICK provides a way to run a script when we encounter
an anomaly and which could be one way to go. A Telegraf instance can be used to
scrape metrics off a cassandra server. Cassandra exposes these metrics via JMX
protocol. The Telegraf instance will inject the metrics into an Influxdb
instance and a Kapacitor instance will process the metrics and take a call
whether we need to spawn another cassandra node or kill one of those. The
process to spawn a new VM will be highly dependent on the cloud platform
provider. Kapacitor provides a plugin for AWS but nothing for Azure or GCP.

Ideally the cloud platform will provide an API which can be utilized by the
Kapacitor tickscript to spawn a new VM. We would need to configure the VM to run
a bootstrap process to setup the docker repository, docker-compose
configuration, pull the docker image and run it. Alternatively, we could setup a
custom VM image using Packer and upload it to AWS as an AMI image which does all
of the bootstrap process as part of the bootup process and joins a cluster. The
bootstrap process could pull the configuration from an API. The bootstrap
process needs to be setup to retry or to try another bootstrap server in case
the configuration management server is down or unavailable.


### Footnotes

[1] We choose running to run cassandra in containerized environment as opposed
to installing cassandra on bare metal. This allows more flexibility in managing
the versions. The performance impact of running cassandra in containers as
opposed to bare metal is insignificant as compared to the benefits obtained by
using containers.

[2] Technically, it should be possible to run both HAProxy and keepalived from a
single docker container but that goes against the docker philosophy of running a
single app within a container.
