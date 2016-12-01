wildfly-jgroups-aws-ping
========================

A Wildfly modules distribution that adds [`AWS_PING`](https://github.com/meltmedia/jgroups-aws) support for the JGroups subsystem.

By adding this module to your Wildfly application server and configuring Wildfly's JGroups subsystem, you can run a Wildfly cluster on AWS EC2. Unlike other approaches using S3_PING or JDBC_PING, using the module imposes no additional dependencies for cluster operation -- it relies only on EC2 metadata tags to identify cluster members.

Installation
-------------

If you have a working Maven build environment, you can build from source using `mvn install`. If you don't want to bother with that see the [releases](https://github.com/soulwing/wildfly-jgroups-aws-ping/releases) page to download a binary distribution. Please verify the PGP signature using my [public key](https://keybase.io/ceharris).

To install the module, simply untar or unzip it into your Wildfly distribution. If you set `WILDFLY_HOME` to the base directory for a Wildfly distribution, you could install the module using this command.

```
tar -C $WILDFLY_HOME -xvf target/wildfly-jgroups-aws-ping-1.0.4-modules.tar.gz
```

Using AWS_PING for Wildfly Clustering
-------------------------------------

Wildfly uses JGroups internally as part of its cluster configuration. You can use the AWS_PING protocol to enable Wildfly clustering support on AWS, using the following CLI commands.

```
/subsystem=jgroups/channel=ee:write-attribute(name=stack,value=tcp)
/subsystem=jgroups/stack=tcp/protocol=MPING:remove
/subsystem=jgroups/stack=tcp/protocol=com.meltmedia.jgroups.aws.AWS_PING:add(\
    add-index=0, module=org.jgroups.aws:awsping, properties={\
        tags=${env.AWS_PING_TAGS:Name}, \
        port_number=7600 \
    })
```

* The port number specified here _must_ be the same as that used in `jgroups-tcp` socket binding elsewhere
  in the configuration. If you want to change this port, it would be a really good idea to set a system property 
  or environment variable to the desired port and use an expression for the `port_number` property here and in the
  corresponding socket binding.
* Use the `AWS_PING_TAGS` environment variable to specify tags to match for EC2 instances that should participate 
  in the same cluster. Using this configuration, if you don't specify tags, EC2 instances that share a common value 
  for the `Name` tag will participate in the same cluster.

### AWS_PING with Dockerized Wildfly

If you're running Wildfly in a Docker container on AWS (e.g using the EC2 Container Service), there a little more 
configuration needed. The _external address_ advertised by an AWS_PING node must be an address that can be reached from another EC2 instance. The default behavior, when running in a Docker container is to use the container's IP address, which generally isn't reachable from other EC2 instances.

The IP address that you usually want to use is the private IP address of the EC2 instance that runs the Docker container with your Wildfly instance. Getting the private IP address using the EC2 metadata service is pretty easy. In the startup script for the Docker container that hosts Wildfly, you can get the private IP address of the EC2 instance and put it in an environment variable.

```
JGROUPS_EXTERNAL_ADDRESS=$(curl --silent --show-error --url http://169.254.169.254/latest/meta-data/local-ipv4)
```

If you arrange for this environment variable to be available when you start Wildfly, you can provide the external address to the JGroups subsystem using the following Wildfly configuration.

```
/subsystem=jgroups/stack=tcp/transport=TCP:write-attribute(name=properties,\
    value={\
        external_addr="${env.JGROUPS_EXTERNAL_ADDRESS}" \
    })
```

Additionally, your container will need to have the ports specified in the `jgroups-tcp` and `jgroups-tcp-fd` socket bindings exposed/published by the host. By default these ports are 7600 and 57600, respectively.


Using AWS_PING with JGroups in an Application
---------------------------------------------

If your application uses JGroups directly, you can use the AWS_PING protocol by replacing in your JGroups stack the current discovery protocol (`MPING` or `TCPPING`) with the `AWS_PING` configuration like this:

``` xml
  <protocol type="com.meltmedia.jgroups.aws.AWS_PING" module="org.jgroups.aws:awsping">
    <property name="tags">aws:autoscaling:groupName</property>
  </protocol>
```

You will also need to make the JGroups module available to your application, by adding
module `org.jgroups.aws:awsping` as a dependency in `jboss-deployment-structure.xml`.
