---
layout: post
title: "Encrypted Property Placeholders in Apache Camel"
date: 2015-12-20 15:27:43
categories: camel fuse red hat property placeholder
---
Encrypted property placeholders are a great way to store and use database, API or broker user IDs and passwords. Red Hat JBoss Fuse utilizes [Jasypt][1] Simplified Encryption to encrypt property values.

To use the Jasypt functionality in Red Hat JBoss Fuse you have to export the JASYPT_ENCRYPTION_PASSWORD variable used to encrypt values. The jasypt-encryption and the camel-jasypt features should be installed by default in the standalone install. In Fabric mode your profile needs the camel-jasypt and the jasypt-encryption features installed.

To check if the features are installed on your fuse instance you can run the following command:
{% highlight shell %}
features:list -i | grep jasypt
{% endhighlight %}

Karaf Shell:
![karaf_shell_jasypt.png](/images/karaf_shell_jasypt.png)

