---
synopsis: >
  Presents a set of recommended tools that help to understand the current status of running CAP services.
status: released
---
<!--- Migrated: @external/java/700-observability0-index.md -> @external/java/observability.md -->

# Observability
<style scoped>
  h1:before {
    content: "Java"; display: block; font-size: 60%; margin: 0 0 .2em;
  }
</style>

<div v-html="$frontmatter.synopsis" />


<!-- #### Content -->
<!--- % include _chapters toc="2,3" %} -->


## Logging { #logging}

When tracking down erroneous behaviour, *application logs* often provide useful hints to reconstruct the executed program flow and isolate functional flaws. In addition, they help operators and supporters to keep an overview about the status of a deployed application. In contrast, messages created via [Messages API](indicating-errors#messages) in custom handlers are reflected to the business user who has triggered the request.


### Logging Facade { #logging-facade}

Various logging frameworks for Java have evolved and are widely used in Open Source software. Most prominent are `logback`, `log4j`, and `JDK logging` (`java.util.logging` or briefly `jul`). These well-established frameworks more or less deal with the same problem domain, that is:

- Logging API for (parameterized) messages with different log levels.
- Hierarchical logger components that can be configured independently.
- Separation of log input (messages, parameters, context) and log output (format, destination).

CAP Java SDK seamlessly integrates with Simple Logging Facade for Java ([SLF4J](http://www.slf4j.org)), which provides an abstraction layer for logging APIs. Applications compiled against SLF4J are free to choose a concrete logging framework implementation at deployment time. Most famous libraries have a native integration to SLF4J, but it also has the capability to bridge legacy logging API calls:

<img src="./assets/slf4j.png" width="500px" class="adapt">

### Logging API { #logging-api}

SLF4J API is simple to use. Retrieve a logger object, choose the log method of the corresponding log level and compose a message with optional parameters via Java API:

```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

Logger logger = LoggerFactory.getLogger("my.loggers.order.consolidation");

@After(event = CqnService.EVENT_READ)
public void readAuthors(List<Orders> orders) {
	orders.forEach(order -> {
		logger.debug("Consolidating order {}", order);
		consolidate(order);
	});
	logger.info("Consolidated {} orders", orders.size());
}
```

Some remarks:

* [Logging Configuration](#logging-configuration) shows how to configure loggers individually to control the emitted log messages.
* The API is robust with regards to the passed parameters, that means, no exception is thrown on parameters mismatch or invalid parameters.

::: tip
Prefer *passing parameters* over *concatenating* the message. `logger.info("Consolidating order " + order)` creates the message `String` regardless the configured log level. This can have a negative impact on performance.
:::

::: tip
A `ServiceException` thrown in handler code and indicating a server error (that is, HTTP response code `5xx`) is *automatically* logged as error along with a stacktrace.
:::


### Logging Configuration with Spring Boot { #logging-configuration}

To set up a logging system, a concrete logging framework has to be chosen and, if necessary, corresponding SLF4j adapters.
In case your application runs on Spring Boot and you make use of Spring starter packages, **you most likely don't have to add any explicit dependency**, as the bundle `spring-boot-starter-logging` is part of all Spring Boot starters. It provides `logback` as default logging framework and in addition adapters for the most common logging frameworks (`log4j` and `jul`).

Similarly, no specific log output configuration is required for local development, as per default, log messages are written to console in human-readable form, which contains timestamp, thread, and logger component information. To customize the log output, for instance to add some application-specific information, you can create corresponding configuration files (such as `logback-spring.xml` for logback) to the classpath and Spring will pick it automatically. Consult the documentation of the dedicated logging framework to learn about the configuration file format.

All logs are written which have a log level greater or equal the configured log level of the corresponding logger object.
Following log levels are available:

| Level    | Use case
| :--------| :--------
| `OFF`    | Turns off the logger
| `TRACE`  | Tracks the application flow only
| `DEBUG`  | Shows diagnostic messages
| `INFO`   | Shows important flows of the application (default level)
| `WARN`   | Indicates potential error scenarios
| `ERROR`  | Shows errors and exceptions

With Spring Boot, there are different convenient ways to configure log levels in development scenario, which will be explained in the following section.

#### At Compile Time { #logging-configuration-compiletime}

The log levels can be configured in _application.yaml_ file:

```bash
# Set new default level
logging.level.root: WARN

# Adjust custom logger
logging.level.my.loggers.order.Consolidation: INFO

# Turn off all loggers matching org.springframework.*:
logging.level.org.springframework: OFF
```

Note that loggers are organized in packages, for instance `org.springframework` controls all loggers that match the name pattern `org.springframework.*`.

#### At Runtime with Restart { #logging-configuration-restart}

You can overrule the given logging configuration with a corresponding environment variable, for instance to set loggers in package `my.loggers.order` to `DEBUG` level, add the following environment variable:

```bash
LOGGING_LEVEL_MY_LOGGERS_ORDER = DEBUG
```

and restart the application.
::: tip
Note that Spring normalizes the variable's suffix to lower case, for example, `MY_LOGGERS_ORDER` to `my.loggers.order`, which actually matches the package name. However, configuring a dedicated logger (such as `my.loggers.order.Consolidation`) can not work in general as class names are in camel case typically.
:::

::: tip
On SAP BTP, Cloud Foundry environment, you can add the environment variable with `cf set-env <app name> LOGGING_LEVEL_MY_LOGGERS_ORDER DEBUG`. Don't forget to restart the application with `cf restart <app name>` afterwards. The additional configuration endures an application restart but might be lost on redeployment.
:::

#### At Runtime Without Restart { #logging-configuration-runtime}

If configured, you can use [Spring actuators](https://docs.spring.io/spring-boot/docs/current/reference/html/actuator.html) to view and adjust logging configuration. Disregarding security aspects and provided that the `loggers` actuator is configured as HTTP endpoint on path `/actuator/loggers`, following example HTTP requests show how to accomplish this:

```bash
# retrieve state of all loggers:
curl http://<app-url>/actuator/loggers

# retrieve state of single logger:
curl http://<app-url>/actuator/loggers/my.loggers.oder.consolidation
 {"configuredLevel":null,"effectiveLevel":"INFO"}

# Change logging level:
curl -X POST -H 'Content-Type: application/json' -d '{"configuredLevel": "DEBUG"}'
  http://<app-url>/actuator/loggers/my.loggers.oder.consolidation
```

[Learn more about Spring actuators and security aspects in section **Metrics**.](#metrics){ .learn-more}

#### Predefined Loggers { #predefined-loggers}

CAP Java SDK has useful built-in loggers that help to track runtime behaviour:

| Logger                         | Use case
| :------------------------------| :--------
| `com.sap.cds.security.authentication`  | Logs authentication and user information
| `com.sap.cds.security.authorization`  | Logs authorization decisions
| `com.sap.cds.odata.v2`  | Logs OData V2 request handling in the adapter
| `com.sap.cds.odata.v4`  | Logs OData V4 request handling in the adapter
| `com.sap.cds.handlers`  | Logs sequence of executed handlers as well as lifecycle of RequestContexts and ChangeSetContexts
| `com.sap.cds.persistence.sql` | Logs executed queries such as CQN and SQL statements (w/o parameters)
| `com.sap.cds.persistence.sql-tx` | Logs transactions, ChangeSetContexts, and connection pool
| `com.sap.cds.multitenancy`  | Logs tenant-related events and sidecar communication
| `com.sap.cds.messaging`  | Logs messaging configuration and messaging events
| `com.sap.cds.remote.odata`  | Logs request handling for remote OData calls
| `com.sap.cds.remote.wire`  | Logs communication of remote OData calls
| `com.sap.cds.auditlog`  | Writes audit log events to application log

Most of the loggers are used on DEBUG level by default as they produce quite some log output. It's convenient to control loggers on package level, for example, `com.sap.cds.security` covers all loggers that belong to this package (namely `com.sap.cds.security.authentication` and `com.sap.cds.security.authorization`).

::: tip
Spring comes with its own [standard logger groups](https://docs.spring.io/spring-boot/docs/2.1.1.RELEASE/reference/html/boot-features-logging.html#boot-features-custom-log-groups). For instance, `web` is useful to track HTTP requests. However, HTTP access logs gathered by the Cloud Foundry platform router are also available in the application log.
:::

### Logging Service { #logging-service}

The platform offers a central application logging service to which the application log output can be streamed to. Operators can analyze the log output by means of higher-level tools.

In Cloud Foundry environment, the service `application-logs` offers an ELK stack (Elasticsearch/Logstash/Kibana). To get connected with the logging service, the application needs to be bound against a corresponding service instance. To match the log output format and structure expected by the logging service, it's recommended to use a prepared encoder from [cf-java-logging-support](https://github.com/SAP/cf-java-logging-support) that matches the configured logger framework. `logback` is used by default as outlined in [Logging Frameworks](#logging-configuration):

```xml
<dependency>
	<groupId>com.sap.hcp.cf.logging</groupId>
	<artifactId>cf-java-logging-support-logback</artifactId>
	<version>${logging.support.version}</version>
</dependency>
```

By default, the library appends additional fields to the log output such as correlation id or Cloud Foundry space. To instrument incoming HTTP requests, a servlet filter needs to be created as. See [Instrumenting Servlets](https://github.com/SAP/cf-java-logging-support/wiki/Instrumenting-Servlets) for more details.

During local development, you might want to stick to the (human-readable) standard log line format. This boils down to have different logger configurations for different Spring profiles. The following sample configuration (file `resources/logback-spring.xml`) outlines how you can achieve this. `cf-java-logging-support` is only active for profile `cloud`, since all other profiles are configured with the standard logback output format:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE xml>
<configuration debug="false" scan="false">
	<springProfile name="cloud">
		<!-- logback configuration of ConsoleAppender according
		     to cf-java-logging-support documentation -->
		[...]
	</springProfile>
	<springProfile name="!cloud">
		<include resource="org/springframework/boot/logging/logback/base.xml"/>
	</springProfile>
</configuration>
```

::: tip
For an example on how to set up a multitenant aware CAP Java application with enabled logging service support, have a look at section [Multitenancy > Adding Logging Service Support](./multitenancy#app-log-support).
:::

### Correlation IDs

In general, a request can be handled by unrelated execution units such as internal threads or remote services. This fact makes it hard to correlate the emitted log lines of the different contributors in an aggregated view. The problem can be solved by enhancing the log lines with unique correlation IDs, which are assigned to the initial request and propagated throughout the call tree.

In case you've configured `cf-java-logging-support` as described in [Logging Service](#logging-service) before, *correlation IDs are handled out-of-the-box by the CAP Java SDK*. In particular, this includes:

- Generation of IDs in non-HTTP contexts
- Thread propagation through [Request Contexts](./request-contexts#threading-requestcontext)
- Propagation to remote services when called via CloudSDK (for instance [Remote Services](./remote-services) or [MTX sidecar](./multitenancy#mtx-sidecar-server))

By default, the ID is accepted and forwarded via HTTP header `X-CorrelationID`. If you want to accept `X-Correlation-Id` header in incoming requests alternatively, follow the instructions given in the guide [Instrumenting Servlets](https://github.com/SAP/cf-java-logging-support/wiki/Instrumenting-Servlets#correlation-id).


## Monitoring and Tracing { #monitoring-and-tracing}

Connect your productive application to a [monitoring](#monitoring) tool in order to identify resource bottlenecks at an early stage and to take appropriate countermeasurements.
Sometimes it is necessary to use a [tracing](#tracing) tool that allows much deeper insights to track down resource consumption issues.


### Monitoring { #monitoring}

When connected to a monitoring tool, applications can report information about memory, CPU, and network usage, which forms the basis for resource consumption overview and reporting capabilities.
In addition, call-graphs can be reconstructed and visualized that represent the flow of web requests within the components and services.

On SAP BTP, [Dynatrace](https://www.dynatrace.com/support/help) is positioned as monitoring solution for productive applications and services.
You can add a Dynatrace connection to your CAP Java application by [additional configuration](https://help.sap.com/docs/BTP/65de2977205c403bbc107264b8eccf4b/1610eac123c04d07babaf89c47d82c91.html).

### Tracing { #tracing}

To minimize overhead at runtime, monitoring information is gathered rather on a global application level and hence might not be sufficient to troubleshoot specific issues. In such a situation, the use of more focused tracing tools can be an option. Typically, such tools are capable of focusing a specific aspect of an application (for instance JVM Garbage Collection), but they come with an additional overhead and therefore shouln't be constantly active. Hence, they need to meet following requirements:

* Switchable at runtime
* Use a communication channel not exposed to unauthorized users
* Not interfering or even blocking business requests

How can dedicated Java tracing tools access the running services in a secure manner? The depicted diagram shows recommended options that **do not require exposed HTTP endpoints**:

<img src="./assets/remote-tracing.png" width="600px" class="adapt">

As authorized operator, you can access the container and start tools [locally](#tracing-local) in a CLI session running with the same user as the target process. Depending on the protocol, the JVM supports on-demand connections (for example, JVM diagnostic tools such as `jcmd`). Alternatively, additional JVM configuration is required as prerequisite (JMX).
A bunch of tools also support [remote](#tracing-remote) connections in a secure way. Instead of running the tool locally, a remote daemon is started as a proxy in the container, which connects the JVM with a remote tracing tool via an ssh tunnel.

#### Local Tools { #tracing-local}

Various CLI-based tools for JVMs are delivered with the SDK. Popular examples are [diagnostic tools](https://docs.oracle.com/javase/8/docs/technotes/guides/troubleshoot/toc.html) such as `jcmd`, `jinfo`, `jstack`, and `jmap`, which help to fetch basic information about the JVM process regarding all relevant aspects. You can take stack traces, heap dumps, fetch GC events and read Java properties etc.
The SAP JVM comes with additional handy tracing tools: `jvmmon` and `jvmprof`. The latter for instance provides a helpful set of traces that allow a deep insight into JVM resource consumption. The collected data is stored within a `prf`-file and can be analyzed offline in the [SAP JVM Profiler frontend](https://wiki.scn.sap.com/wiki/display/ASJAVA/Features+and+Benefits).

#### Remote Profiler Tools { #tracing-remote}

It's even more convenient to interact with the JVM with a frontend client running on a local machine. As already mentioned, a remote daemon as the endpoint of an ssh tunnel is required. Some representative tools are:

- [SAP JVM Profiler](https://wiki.scn.sap.com/wiki/display/ASJAVA/Features+and+Benefits) for SAP JVM with [Memory Analyzer](https://www.eclipse.org/mat/) integration. Find a detailed documentation how to set up a secure remote connection on [Profiling an Application Running on SAP JVM](https://help.sap.com/products/BTP/65de2977205c403bbc107264b8eccf4b/e7097737709842b7bb1c3b9bf3d688b6.html).

- [JProfiler](https://www.ej-technologies.com/products/jprofiler/overview.html) is popular Java profiler available for different platforms and IDEs.

#### Remote JMX-Based Tools { #tracing-jmx}

Java's standardized framework [Java Management Extensions](https://www.oracle.com/java/technologies/javase/javamanagement.html) (JMX) allows introspection and monitoring of the JVM's internal state via exposed Management Beans (MBeans). MBeans also allow to trigger operations at runtime, for instance setting a logger level. Spring Boot automatically creates a bunch of MBeans reflecting the current [Spring configuration and metrics](#spring-boot-actuators) and offers convenient ways for customization. To activate JMX in Spring, add the property:

```bash
spring.jmx.enabled: true
```

to your application configuration. In addition, to enable remote access add the following JVM parameters to open JMX on a specific port (for example, 5000) in the local container:

```bash
-Djava.rmi.server.hostname=localhost
-Dcom.sun.management.jmxremote
-Dcom.sun.management.jmxremote.port=<port>
-Dcom.sun.management.jmxremote.rmi.port=<port>
-Dcom.sun.management.jmxremote.authenticate=false
-Dcom.sun.management.jmxremote.ssl=false
```

::: warning _❗ Attention_{.warning-title}
Exposing JMX/MBeans via a public endpoint can pose a serious security risk.
:::

To establish a connection with a remote JMX client, first open an ssh tunnel to the application via `cf` CLI as operator user:

```bash
cf ssh -N -T -L <local-port>:localhost:<port> <app-name>
```

Afterwards, connect to `localhost:<local-port>` in the JMX client. Common JMX clients are:

- [JConsole](http://openjdk.java.net/tools/svc/jconsole/), which is part of the JDK delivery.
- [OpenJDK Mission Control](https://github.com/openjdk/jmc), which can be installed separately.


<!--- Migrated: @external/java/700-observability03-availability.md -> @external/java/observabilityavailability.md -->
## Availability { #availability}

This section describes how to set up an endpoint for availability or health check. At first glance, providing such a health check endpoint sounds like a simple task. But some aspects need to be considered:

- Authentication (for example, Basic or OAuth2) increases security but introduces higher configuration and maintenance effort.
- Only low resource consumption can be introduced. In case of a public endpoint only low overhead is accepted to avoid denial-of service attacks.
- Ideally, the health check response shows not only the aggregate status, but also the status of crucial services the application depends on such as the underlying persistence.

### Spring Boot Health Checks { #spring-health-checks}

Conveniently, Spring Boot offers out-of-the box capabilities to report the health of the running application and its components. Spring provides a bunch of health indicators, especially `PingHealthIndicator` (`/ping`) and `DataSourceHealthIndicator` (`/db`). This set can be extended by [custom health indicators](#custom-health-indicators) if necessary, but most probably, **setting up an appropriate health check for your application is just a matter of configuration**.

To do so, first add a dependency to Spring Actuators, which forms the basis for health indicators:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

#### Health Indicators { #health-indicators}

By default, Spring exposes the *aggregated* health status on web endpoint `/actuator/health`, including the result of all registered health indicators. But also the `info` actuator is exposed automatically, which might be not desired for security reasons. It's recommended to **explicitly** control web exposition of actuator components in the application configuration. The following configuration snippet is an example suitable for public visible health check information:

```yaml
management:
  endpoint:
    health:
      show-components: always # shows individual indicators
  endpoints:
    web:
      exposure:
        include: health # only expose /health as web endpoint
  health:
     defaults.enabled: false # turn off all indicators by default
     ping.enabled: true
     db.enabled: true
```

The example configuration makes Spring exposing only the health endpoint with health indicators `db` and `ping`. Other indicators ready for auto-configuration such as `diskSpace` are omitted. All components contributing to the aggregated status are shown individually which helps to understand the reason for overall status `DOWN`.

::: tip
CAP Java SDK replaces default `db` indicator in case of multitenancy with an implementation that includes the status of all tenant databases.
:::

Endpoint `/actuator/health` delivers a response (HTTP response code `200` for up, `503` for down) in JSON format with the overall `status` property (for example, `UP` or `DOWN`) and the contributing components:

```json
{
  "status": "UP",
  "components": {
    "db": {
      "status": "UP"
    },
    "ping": {
      "status": "UP"
    }
  }
}
```

It might be advantageous to expose information on detail level. This is an option only for a [protected](#protected-health-checks) health endpoint:

```yaml
management.endpoint.health.show-details: always
```

::: warning _❗ Attention_{.warning-title}
A public health check endpoint may neither disclose system internal data (for example, health indicator details) nor introduce significant resource consumption (for example, doing synchronous database request).
:::

Find all details about configuration opportunities in [Spring Boot Actuator](https://docs.spring.io/spring-boot/docs/current/reference/html/production-ready-features.html) documentation.

#### Custom Health Indicators { #custom-health-indicators}

In case your application relies on additional, mandatory services not covered by default health indicators, you can add a custom health indicator as sketched in this example:

```java
@Component("crypto")
@ConditionalOnEnabledHealthIndicator("crypto")
public class CryptoHealthIndicator implements HealthIndicator {

    @Autowired
    CryptoService cryptoService;

    @Override
    public Health health() {
        Health.Builder status = cryptoService.isAvailalbe() ?
              Health.up() : Health.down();
        return status.build();
    }
}
```

The custom `HealthIndicator` for the mandatory `CryptoService` is registered by Spring automatically and can be controlled with property `management.health.crypto.enabled: true`.

#### Protected Health Checks { #protected-health-checks}

Optionally, you can configure a protected health check endpoint. On the one hand this gives you higher flexibility with regards to the detail level of the response but on the other hand introduces additional configuration and management efforts (for instance key management).
As this highly depends on the configuration capabilities of the client services, CAP does not come with an auto-configuration. Instead, the application has to provide an explicit security configuration on top as outlined with `ActuatorSecurityConfig` in [Customizing Spring Boot Security Configuration](security#custom-spring-security-config).


<!--- Migrated: @external/java/700-observability04-metrics.md -> @external/java/observabilitymetrics.md -->
## Metrics { #metrics}


Metrics are mainly referring to operational information about various resources of the running application such as HTTP sessions and worker threads, JDBC connections, JVM memory including GC statistics etc. Similar to [health checks](#spring-health-checks), Spring Boot comes with a bunch of built-in metrics based on [Spring Actuator](#spring-boot-actuators) framework.
Actuators form an open framework, which can be enhanced by libraries (see [CDS Actuator](#cds-actuator)) as well as the application (see [Custom Actuators](#custom-actuators)) with additional information.


### Spring Boot Actuators and Metrics { #spring-boot-actuators }

[Spring Boot Actuators](https://docs.spring.io/spring-boot/docs/current/reference/html/production-ready-features.html) are designed to provide a set of out-of-the box supportability features, that help to make your application observable in production.

To add actuator support in your application, add following dependency:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

The following table lists some of the available actuators that might be helpful to understand the internal status of the application:

| Actuator    | Description
| :--------| :--------
| `metrics`    | Thread pools, connection pools, CPU, and memory usage of JVM and HTTP web server
| `beans`    | Information about Spring beans created in the application
| `env`    | Exposes the full Spring environment including application configuration
| `loggers`    | List and modify application loggers

By default, nearly all actuators are active. You can switch off actuators individually in the configuration, for instance:

```yaml
management.endpoint.flyway.enabled=false
```

turns off `flyway` actuator.

Depending on the configuration, exposed actuators can have HTTP or [JMX](https://en.wikipedia.org/wiki/Java_Management_Extensions) endpoints. For security reasons, it's recommended to expose only the `health` actuator as web endpoint as described in [Health Indicators](#health-indicators). All other actuators are recommended for local JMX-based access as described in [JMX-based Tools](#tracing-jmx).


#### CDS Actuator { #cds-actuator }

CAP Java SDK plugs a CDS-specific actuator `cds`. This actuator provides information about:

- The version and commit id of the currently used `cds-services` library
- All services registered in the service catalog
- Security configuration (authentication type etc.)
- Loaded features such as `cds-feature-xsuaa`
- Database pool statistics (requires `registerMbeans: true` in [Hikari pool configuration](./persistence-services#datasource-configuration))


#### Custom Actuators { #custom-actuators }

Similar to [Custom Health Indicators](#custom-health-indicators), you can add application-specific actuators as done in the following example:

```java
@Component
@ConditionalOnClass(Endpoint.class)
@Endpoint(id = "app", enableByDefault = true)
public class AppActuator {
	@ReadOperation
	public Map<String, Object> info() {
		Map<String, Object> info = new LinkedHashMap<>();
		info.put("Version", "1.0.0");
		return info;
	}
}
```
The `AppActuator` bean registers an actuator with name `app` that exposes a simple version string.
