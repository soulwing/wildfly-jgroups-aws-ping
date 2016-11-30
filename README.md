wildfly-jgroups-aws-ping
========================

A Wildfly modules distribution that adds `AWS_PING` support for the JGroups
subsystem.

Installation
-------------

Install build and unpack the module in your Wildfly installation.

Replace in your JGroups stack the current discovery protocol (`MPING` or `TCPPING`) with the `AWS_PING` configuration like this:

``` xml
  <protocol type="com.meltmedia.jgroups.aws.AWS_PING" module="org.jgroups.aws:awsping">
    <property name="tags">aws:autoscaling:groupName</property>
  </protocol>
```
