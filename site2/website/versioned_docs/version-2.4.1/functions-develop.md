---
id: version-2.4.1-functions-develop
title: Develop Pulsar Functions
sidebar_label: Develop functions
original_id: functions-develop
---

This tutorial walks you through how to develop Pulsar Functions.

## Available APIs
In Java and Python, you have two options to write Pulsar Functions. In Go, you can use Pulsar Functions SDK for Go.

Interface | Description | Use cases
:---------|:------------|:---------
Language-native interface | No Pulsar-specific libraries or special dependencies required (only core libraries from Java/Python). | Functions that do not require access to the function [context](#context).
Pulsar Function SDK for Java/Python/Go | Pulsar-specific libraries that provide a range of functionality not provided by "native" interfaces. | Functions that require access to the function [context](#context).

The language-native function, which adds an exclamation point to all incoming strings and publishes the resulting string to a topic, has no external dependencies. The following example is language-native function.

<!--DOCUSAURUS_CODE_TABS-->
<!--Java-->
```Java
public class JavaNativeExclamationFunction implements Function<String, String> {
    @Override
    public String apply(String input) {
        return String.format("%s!", input);
    }
}
```
For complete code, see [here](https://github.com/apache/pulsar/blob/master/pulsar-functions/java-examples/src/main/java/org/apache/pulsar/functions/api/examples/JavaNativeExclamationFunction.java).

<!--Python-->
```python
def process(input):
    return "{}!".format(input)
```
For complete code, see [here](https://github.com/apache/pulsar/blob/master/pulsar-functions/python-examples/native_exclamation_function.py).

<!--END_DOCUSAURUS_CODE_TABS-->

The following example uses Pulsar Functions SDK.
<!--DOCUSAURUS_CODE_TABS-->
<!--Java-->
```Java
public class ExclamationFunction implements Function<String, String> {
    @Override
    public String process(String input, Context context) {
        return String.format("%s!", input);
    }
}
```
For complete code, see [here](https://github.com/apache/pulsar/blob/master/pulsar-functions/java-examples/src/main/java/org/apache/pulsar/functions/api/examples/ExclamationFunction.java).

<!--Python-->
```python
from pulsar import Function

class ExclamationFunction(Function):
  def __init__(self):
    pass

  def process(self, input, context):
    return input + '!'
```
For complete code, see [here](https://github.com/apache/pulsar/blob/master/pulsar-functions/python-examples/exclamation_function.py).

<!--Go-->
```Go
package main

import (
	"context"
	"fmt"

	"github.com/apache/pulsar/pulsar-function-go/pf"
)

func HandleRequest(ctx context.Context, in []byte) error{
	fmt.Println(string(in) + "!")
	return nil
}

func main() {
	pf.Start(HandleRequest)
}
```
For complete code, see [here](https://github.com/apache/pulsar/blob/master/pulsar-function-go/examples/inputFunc.go#L20-L36).

<!--END_DOCUSAURUS_CODE_TABS-->

## Schema registry
Pulsar has a built in [Schema Registry](concepts-schema-registry) and comes bundled with a variety of popular schema types(avro, json and protobuf). Pulsar Functions can leverage existing schema information from input topics and derive the input type. The schema registry applies for output topic as well.

## SerDe
SerDe stands for **Ser**ialization and **De**serialization. Pulsar Functions uses SerDe when publishing data to and consuming data from Pulsar topics. How SerDe works by default depends on the language you use for a particular function.

<!--DOCUSAURUS_CODE_TABS-->
<!--Java-->
When you write Pulsar Functions in Java, the following basic Java types are built in and supported by default:

* `String`
* `Double`
* `Integer`
* `Float`
* `Long`
* `Short`
* `Byte`

To customize Java types, you need to implement the following interface.

```java
public interface SerDe<T> {
    T deserialize(byte[] input);
    byte[] serialize(T input);
}
```

<!--Python-->
In Python, the default SerDe is identity, meaning that the type is serialized as whatever type the producer function returns.

You can specify the SerDe when [creating](functions-deploying.md#cluster-mode) or [running](functions-deploying.md#local-run-mode) functions. 

```bash
$ bin/pulsar-admin functions create \
  --tenant public \
  --namespace default \
  --name my_function \
  --py my_function.py \
  --classname my_function.MyFunction \
  --custom-serde-inputs '{"input-topic-1":"Serde1","input-topic-2":"Serde2"}' \
  --output-serde-classname Serde3 \
  --output output-topic-1
```

This case contains two input topics: `input-topic-1` and `input-topic-2`, each of which is mapped to a different SerDe class (the map must be specified as a JSON string). The output topic, `output-topic-1`, uses the `Serde3` class for SerDe. At the moment, all Pulsar Functions logic, include processing function and SerDe classes, must be contained within a single Python file.

When using Pulsar Functions for Python, you have three SerDe options:

1. You can use the [`IdentitySerde`](https://github.com/apache/pulsar/blob/master/pulsar-client-cpp/python/pulsar/functions/serde.py#L70), which leaves the data unchanged. The `IdentitySerDe` is the **default**. Creating or running a function without explicitly specifying SerDe means that this option is used.
2. You can use the [`PickleSerDe`](https://github.com/apache/pulsar/blob/master/pulsar-client-cpp/python/pulsar/functions/serde.py#L62), which uses Python [`pickle`](https://docs.python.org/3/library/pickle.html) for SerDe.
3. You can create a custom SerDe class by implementing the baseline [`SerDe`](https://github.com/apache/pulsar/blob/master/pulsar-client-cpp/python/pulsar/functions/serde.py#L50) class, which has just two methods: [`serialize`](https://github.com/apache/pulsar/blob/master/pulsar-client-cpp/python/pulsar/functions/serde.py#L53) for converting the object into bytes, and [`deserialize`](https://github.com/apache/pulsar/blob/master/pulsar-client-cpp/python/pulsar/functions/serde.py#L58) for converting bytes into an object of the required application-specific type.

The table below shows when you should use each SerDe.

SerDe option | When to use
:------------|:-----------
`IdentitySerde` | When you work with simple types like strings, Booleans, integers.
`PickleSerDe` | When you work with complex, application-specific types and are comfortable with the "best effort" approach of `pickle`.
Custom SerDe | When you require explicit control over SerDe, potentially for performance or data compatibility purposes.

<!--END_DOCUSAURUS_CODE_TABS-->

### Example
Imagine that you're writing Pulsar Functions that are processing tweet objects, you can refer to the following example of `Tweet` class.

<!--DOCUSAURUS_CODE_TABS-->
<!--Java-->

```java
public class Tweet {
    private String username;
    private String tweetContent;

    public Tweet(String username, String tweetContent) {
        this.username = username;
        this.tweetContent = tweetContent;
    }

    // Standard setters and getters
}
```

To pass `Tweet` objects directly between Pulsar Functions, you need to provide a custom SerDe class. In the example below, `Tweet` objects are basically strings in which the username and tweet content are separated by a `|`.

```java
package com.example.serde;

import org.apache.pulsar.functions.api.SerDe;

import java.util.regex.Pattern;

public class TweetSerde implements SerDe<Tweet> {
    public Tweet deserialize(byte[] input) {
        String s = new String(input);
        String[] fields = s.split(Pattern.quote("|"));
        return new Tweet(fields[0], fields[1]);
    }

    public byte[] serialize(Tweet input) {
        return "%s|%s".format(input.getUsername(), input.getTweetContent()).getBytes();
    }
}
```

To apply this customized SerDe to a particular Pulsar Function, you need to:

* Package the `Tweet` and `TweetSerde` classes into a JAR.
* Specify a path to the JAR and SerDe class name when deploying the function.

The following is an example of [`create`](reference-pulsar-admin.md#create-1) operation.

```bash
$ bin/pulsar-admin functions create \
  --jar /path/to/your.jar \
  --output-serde-classname com.example.serde.TweetSerde \
  # Other function attributes
```

> #### Custom SerDe classes must be packaged with your function JARs
> Pulsar does not store your custom SerDe classes separately from your Pulsar Functions. So you need to include your SerDe classes in your function JARs. If not, Pulsar returns an error.

<!--Python-->

```python
class Tweet(object):
    def __init__(self, username, tweet_content):
        self.username = username
        self.tweet_content = tweet_content
```

In order to use this class in Pulsar Functions, you have two options:

1. You can specify `PickleSerDe`, which applies the [`pickle`](https://docs.python.org/3/library/pickle.html) library SerDe.
2. You can create your own SerDe class. The following is an example.

  ```python
  from pulsar import SerDe

  class TweetSerDe(SerDe):
      def __init__(self, tweet):
          self.tweet = tweet

      def serialize(self, input):
          return bytes("{0}|{1}".format(self.tweet.username, self.tweet.tweet_content))

      def deserialize(self, input_bytes):
          tweet_components = str(input_bytes).split('|')
          return Tweet(tweet_components[0], tweet_componentsp[1])
  ```


<!--END_DOCUSAURUS_CODE_TABS-->

In both languages, however, you can write custom SerDe logic for more complex, application-specific types.

## Context
Both the [Java](#java-sdk-functions), [Python](#python-sdk-functions) and [Go](#go-sdk-functions) SDKs provide access to a **context object** that can be used by a function. This context object provides a wide variety of information and functionality to the function.

* The name and ID of a Pulsar Function.
* The message ID of each message. Each Pulsar message is automatically assigned with an ID.
* The name of the topic to which the message is sent.
* The names of all input topics as well as the output topic associated with the function.
* The name of the class used for [SerDe](#serialization-and-deserialization-serde).
* The [tenant](reference-terminology.md#tenant) and namespace associated with the function.
* The ID of the Pulsar Functions instance running the function.
* The version of the function.
* The [logger object](functions-overview.md#logging) used by the function, which can be used to create function log messages.
* Access to arbitrary [user configuration](#user-configuration) values supplied via the CLI.
* An interface for recording [metrics](functions-metrics.md).
* An interface for storing and retrieving state in [state storage](functions-overview.md#state-storage).

<!--DOCUSAURUS_CODE_TABS-->
<!--Java-->
The {@inject: javadoc:Context:/client/org/apache/pulsar/functions/api/Context} interface provides a number of methods that you can use to access the function [context](#context). The various method signatures for the `Context` interface are listed as follows.

```java
public interface Context {
    Record<?> getCurrentRecord();
    Collection<String> getInputTopics();
    String getOutputTopic();
    String getOutputSchemaType();
    String getTenant();
    String getNamespace();
    String getFunctionName();
    String getFunctionId();
    String getInstanceId();
    String getFunctionVersion();
    Logger getLogger();
    void incrCounter(String key, long amount);
    long getCounter(String key);
    void putState(String key, ByteBuffer value);
    ByteBuffer getState(String key);
    Map<String, Object> getUserConfigMap();
    Optional<Object> getUserConfigValue(String key);
    Object getUserConfigValueOrDefault(String key, Object defaultValue);
    void recordMetric(String metricName, double value);
    <O> CompletableFuture<Void> publish(String topicName, O object, String schemaOrSerdeClassName);
    <O> CompletableFuture<Void> publish(String topicName, O object);
}
```

The following example uses several methods available via the `Context` object.

```java
import org.apache.pulsar.functions.api.Context;
import org.apache.pulsar.functions.api.Function;
import org.slf4j.Logger;

import java.util.stream.Collectors;

public class ContextFunction implements Function<String, Void> {
    public Void process(String input, Context context) {
        Logger LOG = context.getLogger();
        String inputTopics = context.getInputTopics().stream().collect(Collectors.joining(", "));
        String functionName = context.getFunctionName();

        String logMessage = String.format("A message with a value of \"%s\" has arrived on one of the following topics: %s\n",
                input,
                inputTopics);

        LOG.info(logMessage);

        String metricName = String.format("function-%s-messages-received", functionName);
        context.recordMetric(metricName, 1);

        return null;
    }
}
```

<!--Go-->
```
func (c *FunctionContext) GetInstanceID() int {
	return c.instanceConf.instanceID
}

func (c *FunctionContext) GetInputTopics() []string {
	return c.inputTopics
}

func (c *FunctionContext) GetOutputTopic() string {
	return c.instanceConf.funcDetails.GetSink().Topic
}

func (c *FunctionContext) GetFuncTenant() string {
	return c.instanceConf.funcDetails.Tenant
}

func (c *FunctionContext) GetFuncName() string {
	return c.instanceConf.funcDetails.Name
}

func (c *FunctionContext) GetFuncNamespace() string {
	return c.instanceConf.funcDetails.Namespace
}

func (c *FunctionContext) GetFuncID() string {
	return c.instanceConf.funcID
}

func (c *FunctionContext) GetFuncVersion() string {
	return c.instanceConf.funcVersion
}

func (c *FunctionContext) GetUserConfValue(key string) interface{} {
	return c.userConfigs[key]
}

func (c *FunctionContext) GetUserConfMap() map[string]interface{} {
	return c.userConfigs
}
```

<!--END_DOCUSAURUS_CODE_TABS-->

### User config
When you run or update Pulsar Functions created using SDK, you can pass arbitrary key/values to them with the command line with the `--userConfig` flag. Key/values must be specified as JSON. The following function creation command passes a user configured key/value to a function.

```bash
$ bin/pulsar-admin functions create \
  --name word-filter \
  # Other function configs
  --user-config '{"forbidden-word":"rosebud"}'
```

<!--DOCUSAURUS_CODE_TABS-->
<!--Java--> 
The Java SDK [`Context`](#context) object enables you to access key/value pairs provided to Pulsar Functions via the command line (as JSON). The following example passes a key/value pair.

```bash
$ bin/pulsar-admin functions create \
  # Other function configs
  --user-config '{"word-of-the-day":"verdure"}'
```

To access that value in a Java function:

```java
import org.apache.pulsar.functions.api.Context;
import org.apache.pulsar.functions.api.Function;
import org.slf4j.Logger;

import java.util.Optional;

public class UserConfigFunction implements Function<String, Void> {
    @Override
    public void apply(String input, Context context) {
        Logger LOG = context.getLogger();
        Optional<String> wotd = context.getUserConfigValue("word-of-the-day");
        if (wotd.isPresent()) {
            LOG.info("The word of the day is {}", wotd);
        } else {
            LOG.warn("No word of the day provided");
        }
        return null;
    }
}
```

The `UserConfigFunction` function will log the string `"The word of the day is verdure"` every time the function is invoked (which means every time a message arrives). The `word-of-the-day` user config will be changed only when the function is updated with a new config value via the command line.

You can also access the entire user config map or set a default value in case no value is present:

```java
// Get the whole config map
Map<String, String> allConfigs = context.getUserConfigMap();

// Get value or resort to default
String wotd = context.getUserConfigValueOrDefault("word-of-the-day", "perspicacious");
```

> For all key/value pairs passed to Java functions, both the key *and* the value are `String`. To set the value to be a different type, you need to deserialize from the `String` type.

<!--Python-->
In Python function, you can access the configuration value like this.

```python
from pulsar import Function

class WordFilter(Function):
    def process(self, context, input):
        forbidden_word = context.user_config()["forbidden-word"]

        # Don't publish the message if it contains the user-supplied
        # forbidden word
        if forbidden_word in input:
            pass
        # Otherwise publish the message
        else:
            return input
```

The Python SDK [`Context`](#context) object enables you to access key/value pairs provided to Pulsar Functions via the command line (as JSON). The following example passes a key/value pair.

```bash
$ bin/pulsar-admin functions create \
  # Other function configs \
  --user-config '{"word-of-the-day":"verdure"}'
```

To access that value in a Python function:

```python
from pulsar import Function

class UserConfigFunction(Function):
    def process(self, input, context):
        logger = context.get_logger()
        wotd = context.get_user_config_value('word-of-the-day')
        if wotd is None:
            logger.warn('No word of the day provided')
        else:
            logger.info("The word of the day is {0}".format(wotd))
```

<!--END_DOCUSAURUS_CODE_TABS-->

### Logger

<!--DOCUSAURUS_CODE_TABS-->
<!--Java-->
Pulsar Functions that use the [Java SDK](#java-sdk-functions) have access to an [SLF4j](https://www.slf4j.org/) [`Logger`](https://www.slf4j.org/api/org/apache/log4j/Logger.html) object that can be used to produce logs at the chosen log level. The following example logs either a `WARNING`- or `INFO`-level log based on whether the incoming string contains the word `danger`.

```java
import org.apache.pulsar.functions.api.Context;
import org.apache.pulsar.functions.api.Function;
import org.slf4j.Logger;

public class LoggingFunction implements Function<String, Void> {
    @Override
    public void apply(String input, Context context) {
        Logger LOG = context.getLogger();
        String messageId = new String(context.getMessageId());

        if (input.contains("danger")) {
            LOG.warn("A warning was received in message {}", messageId);
        } else {
            LOG.info("Message {} received\nContent: {}", messageId, input);
        }

        return null;
    }
}
```

If you want your function to produce logs, you need to specify a log topic when creating or running the function. The following is an example.

```bash
$ bin/pulsar-admin functions create \
  --jar my-functions.jar \
  --classname my.package.LoggingFunction \
  --log-topic persistent://public/default/logging-function-logs \
  # Other function configs
```

All logs produced by `LoggingFunction` above can be accessed via the `persistent://public/default/logging-function-logs` topic.

<!--Python-->
Pulsar Functions that use the [Python SDK](#python-sdk-functions) have access to a logging object that can be used to produce logs at the chosen log level. The following example function that logs either a `WARNING`- or `INFO`-level log based on whether the incoming string contains the word `danger`.

```python
from pulsar import Function

class LoggingFunction(Function):
    def process(self, input, context):
        logger = context.get_logger()
        msg_id = context.get_message_id()
        if 'danger' in input:
            logger.warn("A warning was received in message {0}".format(context.get_message_id()))
        else:
            logger.info("Message {0} received\nContent: {1}".format(msg_id, input))
```

If you want your function to produce logs on a Pulsar topic, you need to specify a **log topic** when creating or running the function. The following is an example.

```bash
$ bin/pulsar-admin functions create \
  --py logging_function.py \
  --classname logging_function.LoggingFunction \
  --log-topic logging-function-logs \
  # Other function configs
```

All logs produced by `LoggingFunction` above can be accessed via the `logging-function-logs` topic.

<!--Go-->
The following Go Function example shows different log levels based on the function input.

```
import (
	"context"

	"github.com/apache/pulsar/pulsar-function-go/log"
	"github.com/apache/pulsar/pulsar-function-go/pf"
)

func loggerFunc(ctx context.Context, input []byte) {
	if len(input) <= 100 {
		log.Infof("This input has a length of: %d", len(input))
	} else {
		log.Warnf("This input is getting too long! It has {%d} characters", len(input))
	}
}

func main() {
	pf.Start(loggerFunc)
}
```

When you use `logTopic` related functionalities in Go Function, import `github.com/apache/pulsar/pulsar-function-go/log`, and you do not have to use the `getLogger()` context object. 

<!--END_DOCUSAURUS_CODE_TABS-->

## Metrics
Pulsar Functions can publish arbitrary metrics to the metrics interface which can be queried. 

> If a Pulsar Function uses the language-native interface for [Java](functions-api.md#java-native-functions) or [Python](#python-native-functions), that function is not able to publish metrics and stats to Pulsar.

<!--DOCUSAURUS_CODE_TABS-->
<!--Java-->
You can record metrics using the [`Context`](#context) object on a per-key basis. For example, you can set a metric for the `process-count` key and a different metric for the `elevens-count` key every time the function processes a message. 

```java
import org.apache.pulsar.functions.api.Context;
import org.apache.pulsar.functions.api.Function;

public class MetricRecorderFunction implements Function<Integer, Void> {
    @Override
    public void apply(Integer input, Context context) {
        // Records the metric 1 every time a message arrives
        context.recordMetric("hit-count", 1);

        // Records the metric only if the arriving number equals 11
        if (input == 11) {
            context.recordMetric("elevens-count", 1);
        }

        return null;
    }
}
```

> For instructions on reading and using metrics, see the [Monitoring](deploy-monitoring.md) guide.

<!--Python-->
You can record metrics using the [`Context`](#context) object on a per-key basis. For example, you can set a metric for the `process-count` key and a different metric for the `elevens-count` key every time the function processes a message. The following is an example.

```python
from pulsar import Function

class MetricRecorderFunction(Function):
    def process(self, input, context):
        context.record_metric('hit-count', 1)

        if input == 11:
            context.record_metric('elevens-count', 1)
```

<!--END_DOCUSAURUS_CODE_TABS-->

### Access metrics
To access metrics created by Pulsar Functions, refer to [Monitoring](deploy-monitoring.md) in Pulsar. 

## State storage
Pulsar Functions use [Apache BookKeeper](https://bookkeeper.apache.org) as a state storage interface. Pulsar installation, including the local standalone installation, includes deployment of BookKeeper bookies.

Since Pulsar 2.1.0 release, Pulsar integrates with Apache BookKeeper [table service](https://docs.google.com/document/d/155xAwWv5IdOitHh1NVMEwCMGgB28M3FyMiQSxEpjE-Y/edit#heading=h.56rbh52koe3f) to store the `State` for functions. For example, a `WordCount` function can store its `counters` state into BookKeeper table service via Pulsar Functions State API.