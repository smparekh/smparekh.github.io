---
layout: post
title: "Encrypted Property Placeholders in Apache Camel"
date: 2015-12-20 15:27:43
categories: camel fuse red hat property placeholder
---
Encrypted property placeholders are a great way to store and use database, API or broker user IDs and passwords. Red Hat JBoss Fuse utilizes [Jasypt][1] Simplified Encryption to encrypt property values.

To use the Jasypt functionality in Red Hat JBoss Fuse you have to export the `JASYPT_ENCRYPTION_PASSWORD` variable before starting the server. The `jasypt-encryption` and the `camel-jasypt` features should already be installed by default in the standalone Fuse install. In Fabric mode the `camel-jasypt` and the `jasypt-encryption` features need to be part of your profile.

### Export the encryption password
{% highlight bash %}
[fedora@sparekh-dev-6 bin]$ export JASYPT_ENCRYPTION_PASSWORD=s3cr3t
{% endhighlight %}

### Verify Jasypt install on Fuse
To check if the features are installed on your fuse instance you can run the following command in the karaf shell:
{% highlight bash %}
features:list -i | grep jasypt
{% endhighlight %}

{% include figure.html src="/images/karaf_shell_jasypt.png" caption="Checking for Jasypt features in Karaf shell" %}

Or you can also check in the Hawtio webconsole by navigating to the OSGi &rarr; Features tab:
{% include figure.html src="/images/hawtio_jasypt.png" caption="Checking for Jasypt features in Hawtio webconsole" %}

### Use encrypted properties in a Camel route
I have modified the sample blueprint route based on the `camel-blueprint-archetype` to show a working camel route with encrypted properties. The project is located on my GitHub repositories page: [camel-blueprint-encrypted][3]

The changes I made to the vanilla `camel-blueprint-archetype` are explained below.

Before we set up a Camel route to decrypt a property we have to encrypt it. Download the Jasypt v1.9.2 binaries from [SourceForge][2]. After you unzip the project, head over to the bin directory.

Encrypt the word 'Camel' using the `encrypt.sh` script.

Note: Your encrypted output will not be the same as mine.
{% highlight bash %}
[fedora@sparekh-dev-6 bin]$ ./encrypt.sh input="Camel" password=s3cr3t algorithm=PBEWITHMD5ANDDES

----ENVIRONMENT-----------------

Runtime: Oracle Corporation OpenJDK 64-Bit Server VM 25.65-b01



----ARGUMENTS-------------------

algorithm: PBEWITHMD5ANDDES
input: Camel
password: s3cr3t



----OUTPUT----------------------

orS/w2nwBZ7YoJCTqtp54g==

{% endhighlight %}  

There are some extra steps involved if you want to use encrypted properties in your Blueprint Camel routes.

First, add the Jasypt `dependency` and `Import-Package` information to the project pom.
{% highlight xml %}
...
<dependency>
    <groupId>org.apache.servicemix.bundles</groupId>
    <artifactId>org.apache.servicemix.bundles.jasypt</artifactId>
    <version>1.9.2_1</version>
    <scope>runtime</scope>
</dependency>
<dependency>
    <groupId>org.apache.karaf.jaas</groupId>
    <artifactId>org.apache.karaf.jaas.jasypt</artifactId>
    <version>2.4.0.redhat-620133</version>
    <scope>runtime</scope>
</dependency>
...
<!-- to generate the MANIFEST-FILE of the bundle -->
<plugin>
    <groupId>org.apache.felix</groupId>
    <artifactId>maven-bundle-plugin</artifactId>
    <version>2.3.7</version>
    <extensions>true</extensions>
    <executions>
        <execution>
            <id>bundle-manifest</id>
            <phase>process-classes</phase>
            <goals>
                <goal>manifest</goal>
            </goals>
        </execution>
    </executions>
    <configuration>
        <instructions>
            <Bundle-SymbolicName>camel-blueprint-encrypted</Bundle-SymbolicName>
            <Private-Package>com.shaishav.camel.blueprint.test</Private-Package>
            <Import-Package>org.jasypt.encryption.pbe;version=1.9.2, org.jasypt.encryption.pbe.config;version=1.9.2, org.osgi.service.blueprint</Import-Package>
        </instructions>
        </configuration>
</plugin>
...
{% endhighlight %}

Second, replace the plaintext value with the encrypted value in your property file enclosed by `ENC()`
{% highlight text %}
body.text=ENC(orS/w2nwBZ7YoJCTqtp54g==)
{% endhighlight %}

Third, you have to add the `enc` and `cm` namespaces to your Blueprint XML.
{% highlight xml %}
xmlns:cm="http://aries.apache.org/blueprint/xmlns/blueprint-cm/v1.1.0"
xmlns:enc="http://karaf.apache.org/xmlns/jasypt/v1.0.0"
{% endhighlight %}

Fourth, add the `cm` and the `enc` section to your Blueprint XML.
{% highlight xml %}
<cm:property-placeholder id="test.props.placeholder" persistent-id="test.props" update-strategy="none" />

<enc:property-placeholder>
    <enc:encryptor class="org.jasypt.encryption.pbe.StandardPBEStringEncryptor">
        <property name="config">
            <bean class="org.jasypt.encryption.pbe.config.EnvironmentStringPBEConfig">
                <property name="algorithm" value="PBEWithMD5AndDES" />
                <property name="passwordEnvName" value="JASYPT_ENCRYPTION_PASSWORD" />
            </bean>
        </property>
    </enc:encryptor>
</enc:property-placeholder>
{% endhighlight %}

The `cm` section specifies the name of the `.cfg` file used to populate properties. In the above example the Camel route expects a file, `test.props.cfg`, in the `etc/` directory of JBoss Fuse.

The `enc` section configures the decryption settings for the encrypted properties. In the above example the `enc:property-placeholder` is configured with the same settings that we encrypted the property.

Blueprint will now handle decrypting the property at Camel route startup. To verify you can run `mvn clean verify` on the sample project. Make sure you export the `JASYPT_ENCRYPTION_PASSWORD` before running the test as the password is required to decrypt successfully.
{% highlight bash %}
[mel-1) thread #0 - timer://foo] timerToLog                     INFO  The message contains Camel at 2015-12-21 22:14:37
{% endhighlight %}
Full Test Output:
{% highlight bash %}
[fedora@sparekh-dev-6 bin]$ mvn clean verify
...
-------------------------------------------------------
 T E S T S
-------------------------------------------------------
Running com.redhat.camel.blueprint.test.RouteTest
[                          main] CamelBlueprintHelper           INFO  Using Blueprint XML file: /home/sparekh/Repos/camel-blueprint-encrypted/target/classes/OSGI-INF/blueprint/blueprint.xml
[                      Thread-0] RawBuilder                     INFO  Copy thread finished.
[                          main] Activator                      INFO  Camel activator starting
[                          main] Activator                      INFO  Camel activator started
[                          main] BlueprintExtender              INFO  No quiesce support is available, so blueprint components will not participate in quiesce operations
[         Blueprint Extender: 1] BlueprintContainerImpl         INFO  Bundle camel-blueprint-encrypted is waiting for namespace handlers [http://aries.apache.org/blueprint/xmlns/blueprint-cm/v1.1.0, http://karaf.apache.org/xmlns/jasypt/v1.0.0, http://camel.apache.org/schema/blueprint]
[         Blueprint Extender: 1] BlueprintContainerImpl         INFO  Bundle org.apache.karaf.jaas.modules is waiting for namespace handlers [http://aries.apache.org/blueprint/xmlns/blueprint-ext/v1.0.0, http://aries.apache.org/blueprint/xmlns/blueprint-cm/v1.1.0, http://karaf.apache.org/xmlns/jaas/v1.0.0]
[                          main] CamelBlueprintHelper           INFO  Updating ConfigAdmin Configuration PID=test.props, factoryPID=null, bundleLocation=file:pojosr by overriding properties {body.text=ENC(/80CXuZkx1uSPqkhczVr3Q==)}
[                          main] RouteTest                      INFO  ********************************************************************************
[                          main] RouteTest                      INFO  Testing: testRoute(com.redhat.camel.blueprint.test.RouteTest)
[                          main] RouteTest                      INFO  ********************************************************************************
[                          main] RouteTest                      INFO  Skipping starting CamelContext as system property skipStartingCamelContext is set to be true.
[                          main] BlueprintCamelContext          INFO  Apache Camel 2.15.1.redhat-620133 (CamelContext: camel-1) is starting
[                          main] DefaultManagementStrategy      INFO  JMX is disabled
[                          main] BlueprintCamelContext          INFO  AllowUseOriginalMessage is enabled. If access to the original message is not needed, then its recommended to turn this option off as it may improve performance.
[                          main] BlueprintCamelContext          INFO  StreamCaching is not in use. If using streams then its recommended to enable stream caching. See more details at http://camel.apache.org/stream-caching.html
[                          main] BlueprintCamelContext          INFO  Route: timerToLog started and consuming from: Endpoint[timer://foo?period=5000]
[                          main] BlueprintCamelContext          INFO  Total 1 routes, of which 1 is started.
[                          main] BlueprintCamelContext          INFO  Apache Camel 2.15.1.redhat-620133 (CamelContext: camel-1) started in 0.304 seconds
[                          main] MockEndpoint                   INFO  Asserting: Endpoint[mock://result] is satisfied
[mel-1) thread #0 - timer://foo] timerToLog                     INFO  The message contains Camel at 2015-12-21 22:14:37
[                          main] RouteTest                      INFO  ********************************************************************************
[                          main] RouteTest                      INFO  Testing done: testRoute(com.redhat.camel.blueprint.test.RouteTest)
[                          main] RouteTest                      INFO  Took: 1.349 seconds (1349 millis)
[                          main] RouteTest                      INFO  ********************************************************************************
[                          main] BlueprintCamelContext          INFO  Apache Camel 2.15.1.redhat-620133 (CamelContext: camel-1) is shutting down
[                          main] DefaultShutdownStrategy        INFO  Starting to graceful shutdown 1 routes (timeout 10 seconds)
[el-1) thread #1 - ShutdownTask] DefaultShutdownStrategy        INFO  Route: timerToLog shutdown complete, was consuming from: Endpoint[timer://foo?period=5000]
[                          main] DefaultShutdownStrategy        INFO  Graceful shutdown of 1 routes completed in 0 seconds
[                          main] BlueprintCamelContext          INFO  Apache Camel 2.15.1.redhat-620133 (CamelContext: camel-1) uptime 1.389 seconds
[                          main] BlueprintCamelContext          INFO  Apache Camel 2.15.1.redhat-620133 (CamelContext: camel-1) is shutdown in 0.038 seconds
[                          main] BlueprintExtender              INFO  Destroying BlueprintContainer for bundle RouteTest
[                          main] BlueprintExtender              INFO  Destroying BlueprintContainer for bundle org.apache.karaf.jaas.modules
[                          main] BlueprintExtender              INFO  Destroying BlueprintContainer for bundle camel-blueprint-encrypted
[                          main] BlueprintCamelContext          INFO  Apache Camel 2.15.1.redhat-620133 (CamelContext: camel-2) is shutting down
[                          main] BlueprintCamelContext          INFO  Apache Camel 2.15.1.redhat-620133 (CamelContext: camel-2) uptime not started
[                          main] BlueprintCamelContext          INFO  Apache Camel 2.15.1.redhat-620133 (CamelContext: camel-2) is shutdown in 0.005 seconds
[                          main] BlueprintExtender              INFO  Destroying BlueprintContainer for bundle org.apache.aries.blueprint
[                          main] BlueprintExtender              INFO  Destroying BlueprintContainer for bundle org.apache.camel.camel-blueprint
[                          main] BlueprintExtender              INFO  Destroying BlueprintContainer for bundle org.apache.karaf.jaas.config
[                          main] BlueprintExtender              INFO  Destroying BlueprintContainer for bundle org.apache.karaf.jaas.jasypt
[                          main] Activator                      INFO  Camel activator stopping
[                          main] Activator                      INFO  Camel activator stopped
[                          main] CamelBlueprintHelper           INFO  Deleting work directory target/bundles/1450754066802
Tests run: 1, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 11.631 sec

Results :

Tests run: 1, Failures: 0, Errors: 0, Skipped: 0

[INFO]
[INFO] --- maven-bundle-plugin:2.3.7:bundle (default-bundle) @ camel-blueprint-encrypted ---
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time: 27.021 s
[INFO] Finished at: 2015-12-21T22:14:38-05:00
[INFO] Final Memory: 23M/442M
[INFO] ------------------------------------------------------------------------
{% endhighlight %}

Now you can store database, broker or API passwords in property files without them being in plaintext.

Reference Docs: [Red Hat JBoss Fuse 6.2.1 Security Guide](https://access.redhat.com/documentation/en-US/Red_Hat_JBoss_Fuse/6.2.1/html/Security_Guide/FMQSecurityEncryptProperties.html)

[1]: http://jasypt.org/ "Jasypt"
[2]: http://sourceforge.net/projects/jasypt/files/jasypt/jasypt%201.9.2/jasypt-1.9.2-dist.zip/download "SourceForge"
[3]: https://github.com/smparekh/camel-blueprint-encrypted "camel-blueprint-encrypted"
