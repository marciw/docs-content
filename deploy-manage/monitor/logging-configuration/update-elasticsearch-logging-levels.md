---
mapped_pages:
  - https://www.elastic.co/guide/en/elasticsearch/reference/current/logging.html
applies_to:
  deployment:
    ess: all
    ece: all
    eck: all
    self: all
---

# Update Elasticsearch logging levels [logging]

You can use {{es}}'s application logs to monitor your cluster and diagnose issues. If you run {{es}} as a service, the default location of the logs varies based on your platform and installation method:

:::::::{tab-set}

::::::{tab-item} Docker
On [Docker](../../deploy/self-managed/install-elasticsearch-with-docker.md), log messages go to the console and are handled by the configured Docker logging driver. To access logs, run `docker logs`.
::::::

::::::{tab-item} Debian (APT) and RPM
For [Debian](../../deploy/self-managed/install-elasticsearch-with-debian-package.md) and [RPM](../../deploy/self-managed/install-elasticsearch-with-rpm.md) installations, {{es}} writes logs to `/var/log/elasticsearch`.
::::::

::::::{tab-item} macOS and Linux
For [macOS and Linux `.tar.gz`](../../deploy/self-managed/install-elasticsearch-from-archive-on-linux-macos.md) installations, {{es}} writes logs to `$ES_HOME/logs`.

Files in `$ES_HOME` risk deletion during an upgrade. In production, we strongly recommend you set `path.logs` to a location outside of `$ES_HOME`. See [Path settings](../../deploy/self-managed/important-settings-configuration.md#path-settings).
::::::

::::::{tab-item} Windows .zip
For [Windows `.zip`](../../deploy/self-managed/install-elasticsearch-with-zip-on-windows.md) installations, {{es}} writes logs to `%ES_HOME%\logs`.

Files in `%ES_HOME%` risk deletion during an upgrade. In production, we strongly recommend you set `path.logs` to a location outside of `%ES_HOME%``. See [Path settings](../../deploy/self-managed/important-settings-configuration.md#path-settings).
::::::

:::::::
If you run {{es}} from the command line, {{es}} prints logs to the standard output (`stdout`).


## Logging configuration [logging-configuration]

::::{important}
Elastic strongly recommends using the Log4j 2 configuration that is shipped by default.
::::


Elasticsearch uses [Log4j 2](https://logging.apache.org/log4j/2.x/) for logging. Log4j 2 can be configured using the log4j2.properties file. Elasticsearch exposes three properties, `${sys:es.logs.base_path}`, `${sys:es.logs.cluster_name}`, and `${sys:es.logs.node_name}` that can be referenced in the configuration file to determine the location of the log files. The property `${sys:es.logs.base_path}` will resolve to the log directory, `${sys:es.logs.cluster_name}` will resolve to the cluster name (used as the prefix of log filenames in the default configuration), and `${sys:es.logs.node_name}` will resolve to the node name (if the node name is explicitly set).

For example, if your log directory (`path.logs`) is `/var/log/elasticsearch` and your cluster is named `production` then `${sys:es.logs.base_path}` will resolve to `/var/log/elasticsearch` and `${sys:es.logs.base_path}${sys:file.separator}${sys:es.logs.cluster_name}.log` will resolve to `/var/log/elasticsearch/production.log`.

```properties
####### Server JSON ############################
appender.rolling.type = RollingFile <1>
appender.rolling.name = rolling
appender.rolling.fileName = ${sys:es.logs.base_path}${sys:file.separator}${sys:es.logs.cluster_name}_server.json <2>
appender.rolling.layout.type = ECSJsonLayout <3>
appender.rolling.layout.dataset = elasticsearch.server <4>
appender.rolling.filePattern = ${sys:es.logs.base_path}${sys:file.separator}${sys:es.logs.cluster_name}-%d{yyyy-MM-dd}-%i.json.gz <5>
appender.rolling.policies.type = Policies
appender.rolling.policies.time.type = TimeBasedTriggeringPolicy <6>
appender.rolling.policies.time.interval = 1 <7>
appender.rolling.policies.time.modulate = true <8>
appender.rolling.policies.size.type = SizeBasedTriggeringPolicy <9>
appender.rolling.policies.size.size = 256MB <10>
appender.rolling.strategy.type = DefaultRolloverStrategy
appender.rolling.strategy.fileIndex = nomax
appender.rolling.strategy.action.type = Delete <11>
appender.rolling.strategy.action.basepath = ${sys:es.logs.base_path}
appender.rolling.strategy.action.condition.type = IfFileName <12>
appender.rolling.strategy.action.condition.glob = ${sys:es.logs.cluster_name}-* <13>
appender.rolling.strategy.action.condition.nested_condition.type = IfAccumulatedFileSize <14>
appender.rolling.strategy.action.condition.nested_condition.exceeds = 2GB <15>
################################################
```

1. Configure the `RollingFile` appender
2. Log to `/var/log/elasticsearch/production_server.json`
3. Use JSON layout.
4. `dataset` is a flag populating the `event.dataset` field in a `ECSJsonLayout`. It can be used to distinguish different types of logs more easily when parsing them.
5. Roll logs to `/var/log/elasticsearch/production-yyyy-MM-dd-i.json`; logs will be compressed on each roll and `i` will be incremented
6. Use a time-based roll policy
7. Roll logs on a daily basis
8. Align rolls on the day boundary (as opposed to rolling every twenty-four hours)
9. Using a size-based roll policy
10. Roll logs after 256 MB
11. Use a delete action when rolling logs
12. Only delete logs matching a file pattern
13. The pattern is to only delete the main logs
14. Only delete if we have accumulated too many compressed logs
15. The size condition on the compressed logs is 2 GB


```properties
####### Server -  old style pattern ###########
appender.rolling_old.type = RollingFile
appender.rolling_old.name = rolling_old
appender.rolling_old.fileName = ${sys:es.logs.base_path}${sys:file.separator}${sys:es.logs.cluster_name}_server.log <1>
appender.rolling_old.layout.type = PatternLayout
appender.rolling_old.layout.pattern = [%d{ISO8601}][%-5p][%-25c{1.}] [%node_name]%marker %m%n
appender.rolling_old.filePattern = ${sys:es.logs.base_path}${sys:file.separator}${sys:es.logs.cluster_name}-%d{yyyy-MM-dd}-%i.old_log.gz
```

1. The configuration for `old style` pattern appenders. These logs will be saved in `*.log` files and if archived will be in `* .log.gz` files. Note that these should be considered deprecated and will be removed in the future.


::::{note}
Log4j’s configuration parsing gets confused by any extraneous whitespace; if you copy and paste any Log4j settings on this page, or enter any Log4j configuration in general, be sure to trim any leading and trailing whitespace.
::::


Note than you can replace `.gz` by `.zip` in `appender.rolling.filePattern` to compress the rolled logs using the zip format. If you remove the `.gz` extension then logs will not be compressed as they are rolled.

If you want to retain log files for a specified period of time, you can use a rollover strategy with a delete action.

```properties
appender.rolling.strategy.type = DefaultRolloverStrategy <1>
appender.rolling.strategy.action.type = Delete <2>
appender.rolling.strategy.action.basepath = ${sys:es.logs.base_path} <3>
appender.rolling.strategy.action.condition.type = IfFileName <4>
appender.rolling.strategy.action.condition.glob = ${sys:es.logs.cluster_name}-* <5>
appender.rolling.strategy.action.condition.nested_condition.type = IfLastModified <6>
appender.rolling.strategy.action.condition.nested_condition.age = 7D <7>
```

1. Configure the `DefaultRolloverStrategy`
2. Configure the `Delete` action for handling rollovers
3. The base path to the Elasticsearch logs
4. The condition to apply when handling rollovers
5. Delete files from the base path matching the glob `${sys:es.logs.cluster_name}-*`; this is the glob that log files are rolled to; this is needed to only delete the rolled Elasticsearch logs but not also delete the deprecation and slow logs
6. A nested condition to apply to files matching the glob
7. Retain logs for seven days


Multiple configuration files can be loaded (in which case they will get merged) as long as they are named `log4j2.properties` and have the Elasticsearch config directory as an ancestor; this is useful for plugins that expose additional loggers. The logger section contains the java packages and their corresponding log level. The appender section contains the destinations for the logs. Extensive information on how to customize logging and all the supported appenders can be found on the [Log4j documentation](https://logging.apache.org/log4j/2.x/manual/configuration.html).


## Configuring logging levels [configuring-logging-levels]

Log4J 2 log messages include a *level* field, which is one of the following (in order of increasing verbosity):

* `FATAL`
* `ERROR`
* `WARN`
* `INFO`
* `DEBUG`
* `TRACE`

By default {{es}} includes all messages at levels `INFO`, `WARN`, `ERROR` and `FATAL` in its logs, but filters out messages at levels `DEBUG` and `TRACE`. This is the recommended configuration. Do not filter out messages at `INFO` or higher log levels or else you may not be able to understand your cluster’s behaviour or troubleshoot common problems. Do not enable logging at levels `DEBUG` or `TRACE` unless you are following instructions elsewhere in this manual which call for more detailed logging, or you are an expert user who will be reading the {{es}} source code to determine the meaning of the logs.

Messages are logged by a hierarchy of loggers which matches the hierarchy of Java packages and classes in the [{{es}} source code](https://github.com/elastic/elasticsearch/). Every logger has a corresponding [dynamic setting](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-cluster-put-settings) which can be used to control the verbosity of its logs. The setting’s name is the fully-qualified name of the package or class, prefixed with `logger.`.

You may set each logger’s verbosity to the name of a log level, for instance `DEBUG`, which means that messages from this logger at levels up to the specified one will be included in the logs. You may also use the value `OFF` to suppress all messages from the logger.

For example, the `org.elasticsearch.discovery` package contains functionality related to the [discovery](../../distributed-architecture/discovery-cluster-formation/discovery-hosts-providers.md) process, and you can control the verbosity of its logs with the `logger.org.elasticsearch.discovery` setting. To enable `DEBUG` logging for this package, use the [Cluster update settings API](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-cluster-put-settings) as follows:

```console
PUT /_cluster/settings
{
  "persistent": {
    "logger.org.elasticsearch.discovery": "DEBUG"
  }
}
```

To reset this package’s log verbosity to its default level, set the logger setting to `null`:

```console
PUT /_cluster/settings
{
  "persistent": {
    "logger.org.elasticsearch.discovery": null
  }
}
```

Other ways to change log levels include:

1. `elasticsearch.yml`:

    ```yaml
    logger.org.elasticsearch.discovery: DEBUG
    ```

    This is most appropriate when debugging a problem on a single node.

2. `log4j2.properties`:

    ```properties
    logger.discovery.name = org.elasticsearch.discovery
    logger.discovery.level = debug
    ```

    This is most appropriate when you already need to change your Log4j 2 configuration for other reasons. For example, you may want to send logs for a particular logger to another file. However, these use cases are rare.


::::{important}
{{es}}'s application logs are intended for humans to read and interpret. Different versions of {{es}} may report information in these logs in different ways, perhaps adding extra detail, removing unnecessary information, formatting the same information in different ways, renaming the logger or adjusting the log level for specific messages. Do not rely on the contents of the application logs remaining precisely the same between versions.
::::


::::{note}
To prevent leaking sensitive information in logs, {{es}} suppresses certain log messages by default even at the highest verbosity levels. To disable this protection on a node, set the Java system property `es.insecure_network_trace_enabled` to `true`. This feature is primarily intended for test systems which do not contain any sensitive information. If you set this property on a system which contains sensitive information, you must protect your logs from unauthorized access.
::::



## Deprecation logging [deprecation-logging]

{{es}} also writes deprecation logs to the log directory. These logs record a message when you use deprecated {{es}} functionality. You can use the deprecation logs to update your application before upgrading {{es}} to a new major version.

By default, {{es}} rolls and compresses deprecation logs at 1GB. The default configuration preserves a maximum of five log files: four rolled logs and an active log.

{{es}} emits deprecation log messages at the `CRITICAL` level. Those messages are indicating that a used deprecation feature will be removed in a next major version. Deprecation log messages at the `WARN` level indicates that a less critical feature was used, it won’t be removed in next major version, but might be removed in the future.

To stop writing deprecation log messages, set `logger.deprecation.level` to `OFF` in `log4j2.properties` :

```properties
logger.deprecation.level = OFF
```

Alternatively, you can change the logging level dynamically:

```console
PUT /_cluster/settings
{
  "persistent": {
    "logger.org.elasticsearch.deprecation": "OFF"
  }
}
```

Refer to [Configuring logging levels](elasticsearch-log4j-configuration-self-managed.md#configuring-logging-levels).

You can identify what is triggering deprecated functionality if `X-Opaque-Id` was used as an HTTP header. The user ID is included in the `X-Opaque-ID` field in deprecation JSON logs.

```js
{
  "type": "deprecation",
  "timestamp": "2019-08-30T12:07:07,126+02:00",
  "level": "WARN",
  "component": "o.e.d.r.a.a.i.RestCreateIndexAction",
  "cluster.name": "distribution_run",
  "node.name": "node-0",
  "message": "[types removal] Using include_type_name in create index requests is deprecated. The parameter will be removed in the next major version.",
  "x-opaque-id": "MY_USER_ID",
  "cluster.uuid": "Aq-c-PAeQiK3tfBYtig9Bw",
  "node.id": "D7fUYfnfTLa2D7y-xw6tZg"
}
```

Deprecation logs can be indexed into `.logs-deprecation.elasticsearch-default` data stream `cluster.deprecation_indexing.enabled` setting is set to true.


### Deprecation logs throttling [_deprecation_logs_throttling]

Deprecation logs are deduplicated based on a deprecated feature key and x-opaque-id so that if a feature is repeatedly used, it will not overload the deprecation logs. This applies to both indexed deprecation logs and logs emitted to log files. You can disable the use of `x-opaque-id` in throttling by changing `cluster.deprecation_indexing.x_opaque_id_used.enabled` to false, refer to this class [javadoc](https://artifacts.elastic.co/javadoc/org/elasticsearch/elasticsearch/8.17.3/org.elasticsearch.server/org/elasticsearch/common/logging/RateLimitingFilter.html) for more details.


## JSON log format [json-logging]

To make parsing Elasticsearch logs easier, logs are now printed in a JSON format. This is configured by a Log4J layout property `appender.rolling.layout.type = ECSJsonLayout`. This layout requires a `dataset` attribute to be set which is used to distinguish logs streams when parsing.

```properties
appender.rolling.layout.type = ECSJsonLayout
appender.rolling.layout.dataset = elasticsearch.server
```

Each line contains a single JSON document with the properties configured in `ECSJsonLayout`. See this class [javadoc](https://artifacts.elastic.co/javadoc/org/elasticsearch/elasticsearch/8.17.3/org.elasticsearch.server/org/elasticsearch/common/logging/ESJsonLayout.html) for more details. However if a JSON document contains an exception, it will be printed over multiple lines. The first line will contain regular properties and subsequent lines will contain the stacktrace formatted as a JSON array.

::::{note}
You can still use your own custom layout. To do that replace the line `appender.rolling.layout.type` with a different layout. See sample below:
::::


```properties
appender.rolling.type = RollingFile
appender.rolling.name = rolling
appender.rolling.fileName = ${sys:es.logs.base_path}${sys:file.separator}${sys:es.logs.cluster_name}_server.log
appender.rolling.layout.type = PatternLayout
appender.rolling.layout.pattern = [%d{ISO8601}][%-5p][%-25c{1.}] [%node_name]%marker %.-10000m%n
appender.rolling.filePattern = ${sys:es.logs.base_path}${sys:file.separator}${sys:es.logs.cluster_name}-%d{yyyy-MM-dd}-%i.log.gz
```

