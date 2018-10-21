[![Build Status](https://secure.travis-ci.org/vy/log4j2-logstash-layout.svg)](http://travis-ci.org/vy/log4j2-logstash-layout)
[![Maven Central](https://img.shields.io/maven-central/v/com.vlkan.log4j2/log4j2-logstash-layout-parent.svg)](https://search.maven.org/#search%7Cga%7C1%7Cg%3A%22com.vlkan.log4j2%22)
[![License](https://img.shields.io/github/license/vy/log4j2-logstash-layout.svg)](http://www.apache.org/licenses/LICENSE-2.0.txt)

`LogstashLayout` plugin provides an efficient [Log4j 2.x](https://logging.apache.org/log4j/2.x/)
layout with customizable and [Logstash](https://www.elastic.co/products/logstash)-friendly
JSON formatting.

By default, `LogstashLayout` ships the official `JSONEventLayoutV1` stated by
[log4j-jsonevent-layout](https://github.com/logstash/log4j-jsonevent-layout)
Log4j 1.x plugin. Compared to
[JSONLayout](https://logging.apache.org/log4j/2.x/manual/layouts.html#JSONLayout)
included in Log4j 2.x and `log4j-jsonevent-layout`, `LogstashLayout` provides
the following additional features:

- Up to 5x higher [performance](#performance).
- JSON schema structure can be customized. (See `template` and `templateUri` parameters.)
- Timestamp formatting can be customized. (See `dateTimeFormatPattern`
  and `timeZoneId` parameters.)

# Table of Contents

- [Usage](#usage)
- [FAT JAR](#fat-jar)
- [Appender Support](#appender-support)
- [Performance](#performance)
- [Contributors](#contributors)
- [License](#license)

<a name="usage"></a>

# Usage

Add the `log4j2-logstash-layout` dependency to your POM file

```xml
<dependency>
    <groupId>com.vlkan.log4j2</groupId>
    <artifactId>log4j2-logstash-layout</artifactId>
    <version>${log4j2-logstash-layout.version}</version>
</dependency>
```

together with a valid `log4j-core` dependency:

```xml
<dependency>
    <groupId>org.apache.logging.log4j</groupId>
    <artifactId>log4j-core</artifactId>
    <version>${log4j2.version}</version>
</dependency>
```

Below you can find a sample `log4j2.xml` snippet employing `LogstashLayout`.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="warn">
    <Appenders>
        <Console name="CONSOLE" target="SYSTEM_OUT">
            <LogstashLayout dateTimeFormatPattern="yyyy-MM-dd'T'HH:mm:ss.SSSZZZ"
                            templateUri="classpath:LogstashJsonEventLayoutV1.json"
                            prettyPrintEnabled="true"
                            stackTraceEnabled="true"/>
        </Console>
    </Appenders>
    <Loggers>
        <Root level="info">
            <AppenderRef ref="CONSOLE"/>
        </Root>
    </Loggers>
</Configuration>
```

This generates an output as follows:

```json
{
  "exception": {
    "exception_class": "java.lang.RuntimeException",
    "exception_message": "test",
    "stacktrace": "java.lang.RuntimeException: test\n\tat com.vlkan.log4j2.logstash.layout.demo.LogstashLayoutDemo.main(LogstashLayoutDemo.java:11)\n"
  },
  "line_number": 12,
  "class": "com.vlkan.log4j2.logstash.layout.demo.LogstashLayoutDemo",
  "@version": 1,
  "source_host": "varlik",
  "message": "Hello, error!",
  "thread_name": "main",
  "@timestamp": "2017-05-25T19:56:23.370+02:00",
  "level": "ERROR",
  "file": "LogstashLayoutDemo.java",
  "method": "main",
  "logger_name": "com.vlkan.log4j2.logstash.layout.demo.LogstashLayoutDemo"
}
```

`LogstashLayout` is configured with the following parameters:

| Parameter Name | Type | Description |
|----------------|------|-------------|
| `prettyPrintEnabled` | boolean | enables pretty-printer (defaults to `false`) |
| `locationInfoEnabled` | boolean | includes the filename and line number in the output (defaults to `false`) |
| `stackTraceEnabled` | boolean | includes stack traces (defaults to `false`) |
| `emptyPropertyExclusionEnabled` | boolean | exclude empty and null properties (defaults to `true`) |
| `dateTimeFormatPattern` | String | timestamp formatter pattern (defaults to `yyyy-MM-dd'T'HH:mm:ss.SSSZZZ`) |
| `timeZoneId` | String | time zone id (defaults to `TimeZone.getDefault().getID()`) |
| `mdcKeyPattern` | String | regex to filter MDC keys (does not apply to direct `mdc:key` access) |
| `ndcPattern` | String | regex to filter NDC items |
| `template` | String | inline JSON template for generating the output (has priority over `templateUri`) |
| `templateUri` | String | JSON template for generating the output (defaults to `classpath:LogstashJsonEventLayoutV1.json`) |
| `lineSeparator` | String | used to separate log outputs (defaults to `System.lineSeparator()`) |
| `maxByteCount` | int | used to cap the internal `byte[]` buffer (defaults to 0) |
| `threadLocalByteBufferEnabled` | boolean | enables thread-local caching of internal `byte[]` buffer and requires `maxByteCount` to be greater than 0 (defaults to `false`) |

`templateUri` denotes the URI pointing to the JSON template that will be used
while formatting the log events. By default, `LogstashLayout` ships the
[JSONEventLayoutV1](https://github.com/logstash/log4j-jsonevent-layout)
in `LogstashJsonEventLayoutV1.json` within the classpath:

```json
{
  "mdc": "${json:mdc}",
  "ndc": "${json:ndc}",
  "exception": {
    "exception_class": "${json:exceptionClassName}",
    "exception_message": "${json:exceptionMessage}",
    "stacktrace": "${json:exceptionStackTrace:text}"
  },
  "line_number": "${json:sourceLineNumber}",
  "class": "${json:sourceClassName}",
  "@version": 1,
  "source_host": "${hostName}",
  "message": "${json:message}",
  "thread_name": "${json:thread:name}",
  "@timestamp": "${json:timestamp}",
  "level": "${json:level}",
  "file": "${json:sourceFileName}",
  "method": "${json:sourceMethodName}",
  "logger_name": "${json:logger:name}"
}
```

In case of need, you can create your own templates with a structure tailored
to your needs. That is, you can add new fields, remove or rename existing
ones, change the structure, etc. Please note that `templateUri` parameter only
supports `file` and `classpath` URI schemes. 

Below is the list of known template variables that will be replaced while
rendering the JSON output.

| Variable Name | Description |
|---------------|-------------|
| `endOfBatch` | `logEvent.isEndOfBatch()` |
| `exceptionClassName` | `logEvent.getThrown().getClass().getCanonicalName()` |
| `exceptionMessage` | `logEvent.getThrown().getMessage()` |
| `exceptionStackTrace` | `logEvent.getThrown().getStackTrace()` (inactive when `stackTraceEnabled=false`) |
| `exceptionStackTrace:text` | `logEvent.getThrown().printStackTrace()` (inactive when `stackTraceEnabled=false`) |
| `exceptionRootCauseClassName` | the innermost `exceptionClassName` in causal chain |
| `exceptionRootCauseMessage` | the innermost `exceptionMessage` in causal chain |
| `exceptionRootCauseStackTrace[:text]` | the innermost `exceptionStackTrace[:text]` in causal chain |
| `level` | `logEvent.getLevel()` |
| `logger:fqcn` | `logEvent.getLoggerFqcn()` |
| `logger:name` | `logEvent.getLoggerName()` |
| `mdc` | Mapped Diagnostic Context `Map<String, String>` returned by `logEvent.getContextData()` |
| `mdc:<key>` | Mapped Diagnostic Context `String` associated with `key` (`mdcKeyPattern` is discarded) |
| `message` | `logEvent.getFormattedMessage()` |
| `message:json` | if `logEvent.getMessage()` is of type `MultiformatMessage` and supports JSON, its read value, otherwise, `{"message": <formattedMessage>}` object |
| `ndc` | Nested Diagnostic Context `String[]` returned by `logEvent.getContextStack()` |
| `sourceClassName` | `logEvent.getSource().getClassName()` |
| `sourceFileName` | `logEvent.getSource().getFileName()` (inactive when `locationInfoEnabled=false`) |
| `sourceLineNumber` | `logEvent.getSource().getLineNumber()` (inactive when `locationInfoEnabled=false`) |
| `sourceMethodName` | `logEvent.getSource().getMethodName()` |
| `thread:id` | `logEvent.getThreadId()` |
| `thread:name` | `logEvent.getThreadName()` |
| `thread:priority` | `logEvent.getThreadPriority()` |
| `timestamp` | `logEvent.getTimeMillis()` formatted using `dateTimeFormatPattern` and `timeZoneId` |
| `timestamp:millis` | `logEvent.getTimeMillis()` |
| `timestamp:nanos` | `logEvent.getNanoTime()` |

JSON field lookups are performed using the `${json:<variable-name>}` scheme
where `<variable-name>` is defined as `<resolver-name>[:<resolver-key>]`.
Characters following colon (`:`) are treated as the `resolver-key` of
which as of now only supported in forms `mdc:<key>` and
`message:json`.

[Log4j 2.x Lookups](https://logging.apache.org/log4j/2.0/manual/lookups.html)
(e.g., `${java:version}`, `${env:USER}`, `${date:MM-dd-yyyy}`) are supported
in templates too. Though note that while `${json:...}` template variables is
expected to occupy an entire field, that is, `"level": "${json:level}"`, a
lookup can be mixed within a regular text: `"myCustomField": "Hello, ${env:USER}!"`.

See `layout-demo` directory for a sample application using the `LogstashLayout`.

<a name="fat-jar"></a>

# Fat JAR

Project also contains a `log4j2-logstash-layout-fatjar` artifact which
includes all its transitive dependencies in a separate shaded package (to
avoid the JAR Hell) with the exception of `log4j-core`, that you need to
include separately.

This might come handy if you want to use this plugin along with already
compiled applications, e.g., Elasticsearch 5.x, which requires Log4j 2.x.

<a name="appender-support"></a>

# Appender Support

`log4j2-logstash-layout` is all about providing a highly customizable JSON
schema for your logs. Though this does not necessarily mean that all of its
features are expected to be supported by every appender in the market. For
instance, while `prettyPrintEnabled=true` works fine with
[log4j2-redis-appender](/vy/log4j2-redis-appender), it should be turned off
for Logstash's `log4j-json` file input type. (See
[Pretty printing in Logstash](/vy/log4j2-logstash-layout/issues/8) issue.)
Make sure you configure `log4j2-logstash-layout` properly in a way that
is aligned with your appender of preference.

<a name="performance"></a>

# Performance

The source code ships a `LogstashLayout`-vs-[`JsonLayout`](https://logging.apache.org/log4j/2.0/manual/layouts.html#JSONLayout)
(the one shipped by default in Log4j 2.X) [JMH](https://openjdk.java.net/projects/code-tools/jmh/)
benchmark assessing the rendering performance of both plugins. There two
different `LogEvent` profiles are employed:

- **full**: `LogEvent` contains MDC, NDC, and an exception.
- **lite:** `LogEvent` has no MDC, NDC, or exception attachment.

To give an idea, we ran the benchmark with the following settings:

- **CPU:** Intel i7 2.70GHz (x86-64)
- **JVM:** Java HotSpot 1.8.0_161
- **OS:** Xubuntu 18.04.1 (4.15.0-34-generic, x86-64)
- **`LogstashLayout:`** used default settings with the following exceptions:
  - **`stackTraceEnabled`:** `true`
  - **`maxByteCount`:** 1024 * 512
  - **`threadLocalByteBufferEnabled`:** `true`
- **`JsonLayout`:** used in two different flavors
  - **`DefaultJsonLayout`:** default settings
  - **`CustomJsonLayout`:** default settings with an additional `"@version": 1`
    field (this forces instantiation of a wrapper class to obtain the necessary
    Jackson view)

The summary (see [BENCHMARK.txt](BENCHMARK.txt) for all) of the results for the
**single-threaded execution of 1,000 `LogEvent`s** are as follows:

| Profile | Layout              | Throughput (ops/sec) | GC Alloc. Rate (MB/sec) |
| ------- | -------------------:| --------------------:| -----------------------:|
| Lite    | `LogstashLayout`    |                  917 |                     143 |
| Lite    | `DefaultJsonLayout` |                  702 |                      54 |
| Lite    | `CustomJsonLayout`  |                  582 |                     152 |
| Full    | `LogstashLayout`    |                   86 |                      59 |
| Full    | `DefaultJsonLayout` |                   17 |                     482 |
| Full    | `CustomJsonLayout`  |                   17 |                     163 |

As results point out, `log4j2-logstash-layout` is the fastest JSON layout in the town.

<a name="contributors"></a>

# Contributors

- [Eric Schwartz](https://github.com/emschwar)
- [Michael K. Edwards](https://github.com/mkedwards)
- [Yaroslav Skopets](https://github.com/yskopets)

<a name="license"></a>

# License

Copyright &copy; 2017-2018 [Volkan Yazıcı](http://vlkan.com/)

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

   http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
