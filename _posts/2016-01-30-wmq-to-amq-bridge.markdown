---
layout: post
title: "Bridging IBM WebSphere MQ and Apache ActiveMQ with Apache Camel"
date: 2016-01-30 17:20:24
categories: camel fuse red hat ibm websphere mq activemq apache
---

One of the most common use cases for Apache Camel is to link two discrete endpoints. In this post I am demonstrating how to bridge an IBM WebSphere MQ Queue to an Apache ActiveMQ Queue.

The route relies on populating properties from a file named "wmq.to.amq.cfg" configured first in the Blueprint XML.

{% highlight xml %}
<!-- Property Placeholder -->
<cm:property-placeholder persistent-id="wmq.to.amq" update-strategy="reload" />
{% endhighlight %}

For the camel route to be functional connection factories to both the IBM WebSphere MQ and Apache ActiveMQ need to be configured.

{% highlight xml %}
...
<!-- Configure Active MQ connection factory -->
<bean id="amqConnectionFactory" class="org.apache.activemq.ActiveMQConnectionFactory">
    <property name="brokerURL" value="${amq.broker.url}" />
    <property name="userName" value="${amq.username}" />
    <property name="password" value="${amq.password}" />
</bean>

<bean id="jmsConfig" class="org.apache.camel.component.jms.JmsConfiguration">
    <property name="connectionFactory" ref="amqConnectionFactory" />
    <property name="concurrentConsumers" value="10" />
</bean>

<bean id="activemq" class="org.apache.activemq.camel.component.ActiveMQComponent">
    <property name="configuration" ref="jmsConfig" />
</bean>

<!-- Configure IBM WebSphere MQ connection factory -->
<bean id="websphereConnectionFactory" class="com.ibm.mq.jms.MQConnectionFactory">
    <property name="transportType" value="1" />
    <property name="hostName" value="${ibm.mq.host}" />
    <property name="port" value="${ibm.mq.port}" />
    <property name="queueManager" value="${ibm.qm.name}" />
</bean>

<bean id="websphereConfig" class="org.apache.camel.component.jms.JmsConfiguration">
    <property name="connectionFactory" ref="websphereConnectionFactory" />
    <property name="concurrentConsumers" value="10" />
</bean>

<bean id="websphere" class="org.apache.camel.component.jms.JmsComponent">
    <property name="configuration" ref="websphereConfig" />
</bean>
...
{% endhighlight %}

The actual Camel route is simple, it consumes from the "websphere" endpoint and produces to the "activemq" endpoint.

{% highlight xml %}
<camelContext trace="false" id="wmqToAmqContext" xmlns="http://camel.apache.org/schema/blueprint">
	<route id="wmqToAmqBridge">
		<from uri="websphere:queue:{ibm.queue.name}?mapJmsMessage=false" />
		<log message="The message contains ${body}" />
		<to uri="activemq:queue:{amq.queue.name}" />
	</route>
</camelContext>
{% endhighlight %}

The "ibm.queue.name" and "amq.queue.name" properties need to be populated in the properties file.

Of course, a broad consumer can also be defined if one wants to consumer from a set of queues using a wildcard and produce to the another set of queues on the ActiveMQ side. In this case, a JMS header stating the destination consumed from could be used to point to the exact queue the message could be sent to.

A working route is available on my GitHub: [camel-wmq-amq][1]

Note: OSGi/Java bundles are necessary for the IBM WebSphere MQ Classes required by this Camel route. They are specific to the version of WebSphere MQ deployed. They are available on the IBM WebSphere website for download.

[1]: https://github.com/smparekh/camel-wmq-amq "camel-wmq-amq"
