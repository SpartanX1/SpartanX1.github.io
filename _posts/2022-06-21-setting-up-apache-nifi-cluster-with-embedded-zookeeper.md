Setting up Apache Nifi Cluster with Embedded Zookeeper
======================================================

Recently while trying to setup Nifi in cluster mode, I quickly realized that there weren't many clear documentations or articles describing how to do that exactly. Adding to the woes, configuring the right cluster configurations wasn’t an easy task.

Nifi includes an setup of Zookeeper by default. Zookeeper is used to create and manage a cluster of nifi instances running on distributed systems. You can of course, use zookeeper externally, but that is beyond the scope of this article.

Enough said! Let me show you how to do it and become cool among your peers.

Essentially all config files are present under `./conf` directory.

```sh
-bin
-conf
  - nifi.properties
  - zookeeper.properties
  - state-management.xml
```

The three files we need to edit are `nifi.properties`, `zookeeper.properties` and `state-management.xml`

Before we do that, you should know that zookeeper recommends an odd no of number of nodes (nifi instances) to begin with. The minimum no should be 3. This is because zookeeper works based on something called a consensus algorithm to maintain the state/connectivity between the nodes. You can read more about the algorithms [here](https://martinfowler.com/articles/patterns-of-distributed-systems/paxos.html). An odd no of nodes ensures majority.

Let’s say you have setup 3 nodes all having the nifi setup. The following files need to be edited in all 3 nodes.

1.  **nifi.properties**

Assuming you have 3 ips corresponding to the 3 nodes,

```
nifi.state.management.embedded.zookeeper.start=true
nifi.zookeeper.connect.string=ip1:2181,ip2:2181,ip3:2181
nifi.zookeeper.auth.type=default
nifi.remote.input.host=ip1 // current node ip
nifi.remote.input.secure=false
nifi.remote.input.socket.port=9998
nifi.remote.input.http.enabled=true // set true if you want http
nifi.cluster.is.node=true
nifi.cluster.node.address=ip1 // current node ip
nifi.cluster.node.protocol.port=7474
nifi.web.http.host=ip1 // current node ip. use either https or http
nifi.web.http.port=8443
nifi.cluster.load.balance.port=6342
```

You can see multiple ports here, which are internally used by Zookeeper to communicate between instances. You can change them if you want. Another thing to note is that only `nifi.web.http` or `nifi.web.https` can be configured at a time.

2. **zookeeper.properties**

This file contains additional info to be used by zookeeper to know about the servers.

```
server.1=ip1:2888:3888
server.2=ip2:2888:3888
server.3=ip3:2888:3888
clientPort=2181
```

3. **state-management.xml**

In order to maintain the nifi state across instances, we need to provide a new state provider pointing to zookeeper.

```xml
<cluster-provider>
<id>zk-provider</id>
<class>org.apache.nifi.controller.state.providers.zookeeper.ZooKeeperStateProvider</class>
<property name="Connect String">ip1:2181,ip2:2181,ip3:2181</property>
        <property name="Root Node">/nifi</property>
        <property name="Session Timeout">10 seconds</property>
        <property name="Access Control">Open</property>
</cluster-provider>
```

I have set the `Access Control`open to be able to login without an username/pass but you should configure it to use one.

Well now that we have configured all 3 files on all the 3 servers, we still need to tell zookeeper which server is which. To do that create a file called `myid` under `./states/zookeeper` . This file will be used by zookeeper in order to identify which instance is which. The value inside this file should be 1 for first instance, 2 for second and so on.

Once you have edited the properties and created the myid file under individual instances, you can start all instances.

If everything went as planned, you should see cmd output as “Election process started” which is very interesting as I mentioned earlier. You can read about it [here](https://zookeeper.apache.org/doc/current/recipes.html#sc_leaderElection).

Once the election is done, you can open the nifi ui on any of the servers.

That’s it! Thanks for reading.
