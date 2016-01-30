---
layout: post
title: "Bridging IBM WebSphere MQ and Apache ActiveMQ with Apache Camel"
date: 2016-01-30 17:20:24
categories: camel fuse red hat ibm websphere mq activemq apache
---

One of the most common use cases for Apache Camel is to link two discrete endpoints. In this post I am demonstrating how to bridge an IBM WebSphere MQ Queue to an Apache ActiveMQ Queue. The camel route configures the connection factories to both the IBM WebSphere MQ and Apache ActiveMQ.

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
