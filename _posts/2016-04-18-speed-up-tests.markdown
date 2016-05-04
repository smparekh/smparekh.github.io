---
layout: post
title: "Speed Up Apache Camel Unit Tests on Blueprint"
date: 2016-04-18 21:18:06
categories: camel apache unit tests blueprint
---
Apache Camel provides great support for unit testing Camel routes on Blueprint, but testing multiple routes can be slow. By default, Camel will create and shutdown the CamelContext per test in your test class.
For example, a simple CBR Camel route with 6 tests takes ~13s to execute (~17s to build):

{% highlight bash %}
Tests run: 6, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 13.299 sec - in com.shaishav.camel.simple.cbr.test.RouteTest

Results :

Tests run: 6, Failures: 0, Errors: 0, Skipped: 0

[INFO] 
[INFO] --- maven-bundle-plugin:2.3.7:bundle (default-bundle) @ camel-simple-cbr ---
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time: 17.434 s
[INFO] Finished at: 2016-05-03T19:48:51-04:00
[INFO] Final Memory: 25M/282M
[INFO] ------------------------------------------------------------------------
{% endhighlight %}

By overriding the `isCreateCamelContextPerClass` method from `CamelTestSupport` to return true in test class  we can force camel to only create one CamelContext at the beginning instead of one per test.

{% highlight java %}
@Override
public boolean isCreateCamelContextPerClass() {
    return true;
}
{% endhighlight %}

If you override the method to return true CamelTestSupport will automatically reset any MockEndpoint in your routes before each test execution to ensure clean `mock:` endpoints.

Snippet from [CamelTestSupport.java L209](https://github.com/apache/camel/blob/camel-2.15.x/components/camel-test/src/main/java/org/apache/camel/test/junit4/CamelTestSupport.java#L209)

{% highlight java %}
} else {
    // and in between tests we must do IoC and reset mock
    postProcessTest();
    resetMocks();
}
{% endhighlight %}

With the addition of the overriden method to the simple CBR test class, test execution takes ~3s (~7s to build):

{% highlight bash %}
Tests run: 6, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 2.797 sec - in com.shaishav.camel.simple.cbr.test.RouteTest

Results :

Tests run: 6, Failures: 0, Errors: 0, Skipped: 0

[INFO] 
[INFO] --- maven-bundle-plugin:2.3.7:bundle (default-bundle) @ camel-simple-cbr ---
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time: 6.822 s
[INFO] Finished at: 2016-05-03T20:24:36-04:00
[INFO] Final Memory: 25M/282M
[INFO] ------------------------------------------------------------------------
{% endhighlight %}

Camel-Simple-CBR source code: [https://github.com/smparekh/camel-simple-cbr](https://github.com/smparekh/camel-simple-cbr)

Note: When using `AdviceWith` with `isCreateCamelContextPerClass` all `mock:` endpoints might not be reset correctly. You can `@Before` JUnit annotation to reset them:

{% highlight java %}
@EndpointInject(uri = "mock:foo")
private MockEndpoint mockEndpoint;

@Before
public void resetMockEndpoints() {
    mockEndpoint.reset();
}
{% endhighlight %}
