---
id: version-0.3-rules
title: Rules
original_id: rules
---

Rules define (a) how data should be extracted when the instrumented
method is executed and (b) which metrics shall be recorded.
The selection on which methods these actions are applied is done through [scopes](instrumentation/scopes.md).

A highlight of inspectIT is the fact that you are completely free in defining how the data is
extracted. In addition, the extracted data can be made visible outside of the instrumented method
in which it was collected: Data can be configured to be propagated up or down with the call stack,
which is explained in the section [Data Propagation](#data-propagation).

The overall concept of rules is best explained with a simple example which is part of the inspectIT Ocelot default configuration:

```yaml
inspectit:
  instrumentation:
    rules:

      record_method_duration:

        entry:
          method_entry_time:
            action: timestamp_nanos
          method_name:
            action: get_method_fqn

        exit:
          method_duration:
            action: elapsed_millis
            data-input:
              sinceNanos: method_entry_time

        metrics:
          '[method/duration]' : method_duration
```

This example rule named `record_method_duration` measures the duration of the instrumented method and outputs the value using the `method/duration` metric.

As the name states, we define under the `entry` property of the rule which actions are performed on method entry. Similarly, the `exit` property defines what is done when the instrumented method returns. In both sections we collect data.

On entry, we collect the current timestamp in a variable named `method_entry_time` and the name of the currently executed method in `method_name`.
These variables are _data_, their names are referred to as _data keys_. Note that we also define how the data is collected: For `method_entry_time` we invoke the [action](#actions) named `timestamp_nanos` and for `method_name` the one named `get_method_fqn`.

This data is then used on method exit: using the action `elapsed_millis` we compute the time which has passed since `method_entry_time`. Finally, the duration computed this way is used as value for the `method/duration` metric. As shown in the [definition](metrics/custom-metrics.md) of this metric, the collected `method_name` is used as tag for all of its views.

## Data Propagation

As illustrated by the previous example, we can collect any amount of data in both the entry and the exit section of an instrumented method. Each data is hereby identified by its name, the _data key_.
Internally, inspectIT creates a dictionary like Object each time an instrumented method is executed. This object is basically a local variable for the method. Whenever data is written, inspectIT stores the value under the given data key in this dictionary. Similarly, whenever data is read, inspectIT looks it up based on the data key in the dictionary. This dictionary is called _inspectIT context_.

If the inspectIT context was truly implemented as explained above, all data would be only visible in the method where it was collected. This however often is not the desired behaviour.
Consider the following example: you instrument the entry method of your HTTP server and collect the request URL as data there. You now of course want this data to be visible as tag for metrics collected in methods called by your entry point. With the implementation above, the request URL would only be visible within the HTTP entry method.

For this reason the inspectIT context implements _data propagation_. The propagation can happen in two directions:

* **Down Propagation:** Data collected in your instrumented method will also be visible to all methods directly or indirectly called by your method. This behaviour already comes [with the OpenCensus Library for tags](https://opencensus.io/tag/#propagation).
* **Up Propagation:** Data collected in your instrumented method will be visible to the methods which caused the invocation of your method. This means that all methods which lie on the call stack will have access to the data written by your method

Up- and down propagation can also be combined: in this case then the data is attached to the control flow, meaning that it will appear as if its value will be passed around with every method call and return.

The second aspect of propagation to consider is the _level_. Does the propagation happen within each Thread separately or is it propagated across threads? Also, what about propagation across JVM boarders, e.g. one micro service calling another one via HTTP? In inspectIT Ocelot we provide the following two settings for the propagation level.

* **JVM local:** The data is propagated within the JVM, even across thread boarders. The behaviour when data moves from one thread to another is defined through [Special Sensors](instrumentation/special-sensors.md).
* **Global:** Data is propagated within the JVM and even across JVM boarders. For example, when an application issues an HTTP request, the globally down propagated data is added to the headers of the request. When the response arrives, up propagated data is collected from the response headers. Again, this protocol specific behaviour is realized through [Special Sensors](instrumentation/special-sensors.md).

### Defining the Behaviour

The propagation behaviour is not defined on rule level but instead globally based on the data keys under the configuration
property `inspectit.instrumentation.data`. Here are some examples extracted from the default configurations of inspectIT:

```yaml
inspectit:
  instrumentation:
    data:
      # for correlating calls across JVM boarders
      prop_origin_service: {down-propagation: GLOBAL, is-tag: false}
      prop_target_service: {up-propagation: GLOBAL, down-propagation: JVM_LOCAL, is-tag: false}

      #we allow the application to be defined at the beginning and to be down propagated from there
      application: {down-propagation: GLOBAL, is-tag: true}

      #this data will only be visible locally in the method where it is collected
      http_method: {down-propagation: NONE}
      http_status: {down-propagation: NONE}
```

Under `inspectit.instrumentation.data`, the data keys are mapped to their desired behaviour.
The configuration options are the following:

|Config Property|Default| Description
|---|---|---|
|`down-propagation`|`JVM_LOCAL`|Configures if values for this data key propagate down and the level of propagation.
Possible values are `NONE`, `JVM_LOCAL` and `GLOBAL`. If `NONE` is configured, no down propagation will take place.
|`up-propagation`|`NONE`| Configures if values for this data key propagate up and the level of propagation.
Possible values are `NONE`, `JVM_LOCAL` and `GLOBAL`. If `NONE` is configured, no up propagation will take place.
|`is-tag`|`true`|If true, this data will act as a tag when metrics are recorded. This does not influence propagation, e.g. typically you want tags to be down propagated JVM locally.

Note that you are free to use data keys without explicitly defining them in the `inspectit.instrumentation.data` section. In this case simply all settings are assumed to be default, which corresponds to the behaviour of OpenCensus tags.

### Interaction with OpenCensus Tags

As explained previously, our inspectIT context can be seen as a more flexible variation of OpenCensus tags. In fact, we designed the inspectIT context so that it acts as a superset of the OpenCensus TagContext.

Firstly, when an instrumented method is entered, a new inspectIT context is created. At this point, it imports any tag values published by OpenCensus directly as data. This also includes the [common tags](metrics/common-tags.md) created by inspectIT. This means, that you can simply read (and overwrite) values for common tags such as `service` or `host_address` at any rule.

The integration is even deeper if you [configured the agent to also extract the metrics from manual instrumentation in your application](configuration/open-census-configuration.md).
Firstly, if a method instrumented by inspectIT Ocelot is executed within a TagContext opened by your application,
these application tags will also be visible in the inspectIT context. Secondly, after the execution of the entry phase of each rule, a new TagContext is opened making the tags written there accessible to metrics collected by your application. Hereby, only data for which down propagation was configured to be `JVM_LOCAL` or greater and for which `is-tag` is true will be visible as tags.

## Actions

Actions are the tool for extracting arbitrary data from your application or the context.
They are effectively Lambda-like functions you can invoke from the entry and the exit phase of rules. They are defined by (a) specifying their input parameters and (b) giving a Java code snippet which defines how the result value is computed from these.

Again, this is best explained by giving some simple examples extracted from inspectIT Ocelot default configuration:

```yaml
inspectit:
  instrumentation:
    actions:

      #computes a nanosecond-timestamp as a long for the current point in time
      timestamp_nanos:
        value: "new Long(System.nanoTime())"

      #computes the elapsed milliseconds as double since a given nanosecond-timestamp
      elapsed_millis:
        input:
          #the timestamp captured via System.nanoTime() to compare against
          sinceNanos: long
        value: "new Double( (System.nanoTime() - sinceNanos) * 1E-6)"

      string_replace_all:
        input:
          regex: String
          replacement: String
          string: String
        value: string.replaceAll(regex,replacement)

      get_method_fqn:
        input:
          _methodName: String
          _class: Class
        value: new StringBuilder(_class.getName()).append('.').append(_methodName).toString()
```

The names of the first two actions, `timestamp_nanos` and `elapsed_millis` should be familiar for you from the initial example in the [rules section](instrumentation/rules.md).

The code executed when a action is invoked is defined through the `value` configuration property. In YAML, this is simply a string. InspectIT however will interpret this string as a Java expression to evaluate. The result value of this expression will be used as result for the action invocation.

Note that the code will not be interpreted at runtime, but instead inspectIT Ocelot will compile the expression to bytecode to ensure maximum efficiency. As indicated by the manual primitive boxing performed for `timestamp_nanos` the compiler has some restrictions. For example Autoboxing is not supported. However, actions are expected to return Objects, therefore manual boxing has to be performed. Under the hood, inspectIT uses the [javassist](http://www.javassist.org/) library, where all imposed restrictions can be found.
The most important ones are that neither Autoboxing, Generics, Anonymous Classes or Lambda Expressions are supported.

After actions have been compiled, they are placed in the same class loader as the class you instrument with them. This means that they can access any class that your application class could also access.

> Even if your action terminates with an exception or error, inspectIT will make sure that this does not affect your application. InspectIT will print information about the error and the faulting action. The execution of the action in the rule where the failure occured will be disabled until you update your configuration.

### Input Parameters

As previously mentioned actions are also free to define any kind of _input parameters_ they need. This is done using the `input` configuration property.
This property maps the names of the input parameters to their expected Java type.
For example, the `elapsed_millis` action declares a single input variable named `sinceNanos` which has the type `long`. Note that for input parameters automatic primitive unboxing is supported.

Another example where the action even defines multiple inputs is `string_replace_all`. Guess what this action does? [Hint](https://docs.oracle.com/javase/8/docs/api/java/lang/String.html#replaceAll-java.lang.String-java.lang.String).

The fourth example shown above is `get_method_fqn`, which uses the _special_ input parameters `_methodName` and `_class`. The fact that these variables are special is indicated by the leading underscore. When normally invoking actions from rules, the user has to take care that all input parameters are assigned a value. For special input parameters inspectIT automatically assigned the desired value. This means that for example `get_method_fqn` can be called without manually assigning any parameter, like it was done in the initial example in the [rules section](instrumentation/rules.md). An overview of all available special input parameters is given below:

|Parameter Name|Type| Description
|---|---|---|
|`_methodName`| [String](https://docs.oracle.com/javase/8/docs/api/java/lang/String.html)| The name of the instrumented method within which this action is getting executed.
|`_class`| [Class](https://docs.oracle.com/javase/8/docs/api/java/lang/Class.html)| The class declaring the instrumented method within which this action is getting executed.
|`_parameterTypes`| [Class](https://docs.oracle.com/javase/8/docs/api/java/lang/Class.html)[]| The types of the parameters which the instrumented method declares for which the action is executed.
|`_this`| (depends on context) | The this-instance in the context of the instrumented method within which this action is getting executed.
|`_args`| [Object](https://docs.oracle.com/javase/8/docs/api/java/lang/Object.html)[] | The arguments with which the instrumented method was called within which this action is getting executed. The arguments are boxed if necessary and packed into an array.
|`_arg0,_arg1,...,_argN`| (depends on context)| The N-th argument with which the instrumented method was called within which this action is getting executed.
|`_returnValue`| (depends on context) | The value returned by the instrumented method within which this action is getting executed. If the method terminated with an exception or the action is executed in the entry phase this is `null`.
|`_thrown`| [Throwable](https://docs.oracle.com/javase/8/docs/api/java/lang/Throwable.html)| The exception thrown by the instrumented method within which this action is getting executed. If the method returned normally or the action is executed in the entry phase this is `null`.
|`_context`| [InspectitContext](https://github.com/inspectIT/inspectit-ocelot/blob/master/inspectit-ocelot-bootstrap/src/main/java/rocks/inspectit/ocelot/bootstrap/exposed/InspectitContext.java) | Gives direct read and write access to the current [context](#data-propagation). Can be used to implement custom data propagation.
|`_attachments`| [ObjectAttachments](https://github.com/inspectIT/inspectit-ocelot/blob/master/inspectit-ocelot-bootstrap/src/main/java/rocks/inspectit/ocelot/bootstrap/exposed/ObjectAttachments.java) | Allows you to attach values to objects instead of to the control flow, as done via `_context`.


### Multiple statements and Imports

Actions can easily become more complex, so that a single expression is not sufficient for expressing the functionality.
For this purpose we introduced the `value-body` configuration property for actions as an alternative to `value`.
`value-body` allows you to specify a Java method body which returns the result of the action. The body is given without surrounding curly braces. One example action from the default configuration making use of this is given below:

```yaml
inspectit:
  instrumentation:
    actions:
      get_servlet_request_path:
        imports:
          - javax.servlet
          - javax.servlet.http
        input:
          _arg0: ServletRequest
        value-body: |
          if(_arg0 instanceof HttpServletRequest) {
            return java.net.URI.create(((HttpServletRequest)_arg0).getRequestURI()).getPath();
          }
          return null;
```

This action is designed to be applied on the Servlet API [doFilter](https://javaee.github.io/javaee-spec/javadocs/javax/servlet/Filter.html#doFilter-javax.servlet.ServletRequest-javax.servlet.ServletResponse-javax.servlet.FilterChain) and
[service](https://javaee.github.io/javaee-spec/javadocs/javax/servlet/Servlet.html#service-javax.servlet.ServletRequest-javax.servlet.ServletResponse) methods.
 It's purpose is to extract HTTP path, however in the servlet API it is not guaranteed that the `ServletRequest` is a `HttpServletRequest`.
 For this reason the action performs an instance-of check only returning the HTTP path if it is available, otherwise `null`.

Normally, all non `java.lang.*` types have to be referred to using their fully qualified name, as done for `java.net.URI` in the example above. However, just like in Java you can import packages using the `import` config option. In this example this allows us to refer to `ServletRequest` and `HttpServletRequest` without using the fully qualified name.

## Defining Rules

Rules glue together [scopes](instrumentation/scopes.md) and [actions](instrumentation/rules.md#actions) to define which actions you want to perform on which application methods.

As you might have noticed, the initial example rule shown in the [rules section](instrumentation/rules.md) did not define any reference to a scope. This is because this rule originates form the default configuration of inspectIT Ocelot, where we don't know yet of which methods you want to collect the response time. Therefore this rule is defined without scopes, but you can easily add some in your own configuration files:

```yaml
inspectit:
  instrumentation:
    rules:

      record_method_duration:
        scopes:
          my_first_scope: true
          my_second_scope: true
```

With this snippet we defined that the existing rule `record_method_duration` gets applied on the two scopes named `my_first_scope` and `my_second_scope`. The `scopes` configuration option maps scope names to `true` or `false`. The rule will be applied on all methods matching any scope where the value is `true`.

Rules define their action within three _phases_:

* **Entry Phase:** The actions defined in this phase get invoked directly before the body of the instrumented method. You can imagine that these actions are "inlined" at the very top of every method instrumented by the given rule.

* **Exit Phase:** The actions defined in this phase get invoked after the body of the instrumented method has terminated. This can be the method either returning normally or throwing an exception. You can imagine that these actions are placed in a `finally` block of a `try` block surrounding the body of the instrumented method.

* **Metrics Phase:** These actions are executed directly after the _exit phase_.
Here, only values for metrics are recorded. No actions will be executed here.

The actions performed in this phases are defined in rules under the `entry`, `exit` and `metrics` configuration options. In the entry and in the exit phase the actions you perform are invocations of [actions](#actions). Please see the [Invoking Actions](#invoking-actions) section for information on how this is done.

In the _metrics phase_ you only can collect metrics, this is explained in the [Collecting Metrics](#collecting-metrics) section.

### Invoking Actions

In this section you will find out how to collect data in the entry and exit phase of rules by invoking [actions](#actions) and storing the results in the [inspectIT context](#data-propagation).

Let's take a look again at the entry phase definitions of the ``record_method_duration`` rule:

```yaml
#inspectit.instrumentation.rules is omitted here
record_method_duration:
  entry:
    method_entry_time:
      action: timestamp_nanos
    method_name:
      action: get_method_fqn
```

The `entry` and `exit` configuration options are YAML dictionaries mapping data keys to _action invocations_.
This means the keys used in the dictionaries define the data key for which a value is being defined. Correspondingly, the assigned value defines which action is invoked to define the value of the data key.

In the example above `method_entry_time` is a data key. The action which is invoked is defined through the `action` configuration option. In this case, it is the action named `timestamp_nanos`.

#### Assigning Input Parameter Values

Actions [can require input parameters](#input-parameters) which need to be assigned when invoking them.
There are currently two possible ways of doing this:

* **Assigning Data Values:** In this case, the value for a given data key is extracted from the [inspectIT context](#data-propagation) and passed to the action
* **Assigning Constant Values:** In this case a literal specified in the configuration will directly be passed to the action.

We have already seen how the assignment of data values to parameters is done in the exit phase of the `record_method_duration` rule:

```yaml
#inspectit.instrumentation.rules is omitted here
record_method_duration:
  exit:
    method_duration:
      action: elapsed_millis
      data-input:
        sinceNanos: method_entry_time
```

The `elapsed_millis` action requires a value for the input parameter `sinceNanos`.
In this example we defined that the value for the data key `method_entry_time` is used for `sinceNanos`.

The assignment of constant values works very similar:

```yaml
#inspectit.instrumentation.rules is omitted here
example_rule:
  entry:
    hello_world_text:
      action: set
      constant-input:
        value: "Hello World!"
```

Note that when assigning a constant value, inspectIT Ocelot automatically converts the given value to the type expected by the action. This is done using the [Spring Conversion Service](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/core/convert/ConversionService.html). For example, if your action expects a parameter of type `java.time.Duration`, you can simply pass in `"42s"` as constant.

As you might have noticed, `data-input` and `constant-input` are again YAML dictionaries.
This allows you to assign values for actions which expect multiple input parameters.
You can also mix which parameters you assign from data and which from constants:


```yaml
#inspectit.instrumentation.rules is omitted here
example_rule:
  entry:
    bye_world_text:
      action: string_replace_all
      data-input:
        string: hello_world_text
      constant-input:
        regex: "Hello"
        replacement: "Bye"
```

As expected given the [definition](#actions) of the `string_replace_all` action, the value of `bye_world_text` will be `"Bye World!"`

#### Adding Conditions

It is possible to add conditions to action invocations. The invocation will only occur if the specified condition is met. Currently, the following configuration options can be used:

|Config Option| Description
|---|---|
|`only-if-null`| Only executes the invocation if the value assigned with the given data key is null.
|`only-if-not-null`| Only executes the invocation if the value assigned with the given data key is not null.
|`only-if-true`| Only executes the invocation if the value assigned with the given data key is the boolean value `true`.
|`only-if-false`| Only executes the invocation if the value assigned with the given data key is the boolean value `false`.


An example for the usage of a condition is given below:

```yaml
#inspectit.instrumentation.rules is omitted here
example_rule:
  entry:
    application_name:
      action: set
      constant-input:
        value: "My-Application"
    only-if-null: application_name
```

In this example we define an invocation to set the value of the data key `application_name`
to `"My-Application"`. However, this assignment is only performed if `application_name` previously was null, meaning that no value has been assigned yet. This mechanism is in particular useful when `application_name` is [down propagated](#data-propagation).

If multiple conditions are given for the same action invocation, the invocation is only executed if *all* conditions are met.

#### Execution Order

As we can use data values for input parameters and for conditions, action invocations can depend on another. This means that a defined order on action executions within each phase is required for rules to work as expected.

As all invocations are specified under the `entry` or the `exit` config options which are YAML dictionaries, the order they are given in the config file does not matter. YAML dictionaries do not maintain or define an order of their entries.

However, inspectIT Ocelot _automatically_ orders the invocations for you correctly.
For each instrumented method the agent first finds all rules which have scopes matching the given method. Afterwards, these rules get combined into one "super"-rule by simply merging the `entry`, `exit` and `metrics` phases.

Within the `entry` and the `exit` phase, actions are now automatically ordered based on their dependencies. E.g. if the invocation writing `data_b` uses `data_a` as input, the invocation writing `data_a` is guaranteed to be executed first! Whenever you use a data value as value for a parameter or in a condition, this will be counted as a dependency.

In some rare cases you might want to change this behaviour. E.g. in tracing context you want to store the [down propagated](#data-propagation) `span_id` in `parent_span`, before the current method assigns a new `span_id`. This can easily be realized using the `before` config option for action invocations:

```yaml
#inspectit.instrumentation.rules is omitted here
example_rule:
  entry:
    parent_span:
      action: set
      data-input:
        value: span_id
    before:
      span_id: true
```

### Collecting Metrics

Metrics collection is done in the metrics phase of a rule, which can be configured using the `metrics` option:

```yaml
#inspectit.instrumentation.rules is omitted here
example_rule:
  #...
  exit:
    method_duration:
      #action invocation here....

  metrics:
    '[method/duration]' : method_duration
    '[some/other/metric]' : 42
```

The metrics phase is executed after the exit phase of the rule. As shown above, you can simply assign values to metrics based on their name. You must however have [defined the metric](metrics/custom-metrics.md) to use them.

The measurement value written to the metric can be specified by giving a data key. This was done in the example above for `method/duration`: Here, the value for the data key `method_duration` is taken, which we previously wrote in the exit phase.
Alternatively you can just specify a constant which will be used, like it was done for `some/other/metric`.

If the value assigned with the data key you specified is `null` (e.g. no data was collected), no value for the metric will be written out.

In addition, all configured tags for the metrics will also be taken from the inspectIT context, if they have been [configured to be used as tags](#defining-the-behaviour).

### Collecting Traces

The inspectIT Ocelot agent allows you to record method invocations as [OpenCensus spans](https://opencensus.io/tracing/span/).
In order to make your collected spans visible, you must first set up a [trace exporter](tracing/trace-exporters.md).

Afterwards you can define that all methods matching a certain rule will be traced:

```yaml
inspectit:
  instrumentation:
    rules:
      example_rule:
        tracing:
          start-span: true
```

For example, using the previous configuration snippet, each method that matches the scope definition of the `example_rule` rule will appear within a trace. Its appearance can be customized using the following properties which can be set in the rule's `tracing` section.

|Property |Default| Description
|---|---|---|
|`start-span`|`false`|If true, all method invocations of methods matching any scope of this rule will be collected as spans.
|`name`|`null`|Defines a data key whose value will be used as name for the span. If it is `null` or the value for the data key is `null`, the full qualified name of the method will be used. Note that the value for the data key must be written in the entry section of the rule at latest!
|`kind`|`null`|Can be `null`, `CLIENT` or `SERVER` corresponding to the [OpenCensus values](https://opencensus.io/tracing/span/kind/).
|`attributes`|`{}` (empty dictionary) |Maps names of attributes to data keys whose values will be used on exit to populate the given attributes.

Commonly, you do not want to have the full qualified name of the instrumented method as span name. For example, for HTTP requests you typically want the HTTP path as span name. This behaviour can be customized using the `name` property:

```yaml
inspectit:
  instrumentation:
    rules:
      servlet_api_service:
        tracing:
          start-span: true
          name: http_path
        entry:
          http_path:
           #... action call to fetch the http path here
```

> The name must exist at the end of the entry section and cannot be set in the exit section.

It is often desirable to not capture every trace, but instead [sample](https://opencensus.io/tracing/sampling/) only a subset.
This can be configured using the `sample-probability` setting under the `tracing` section:

```yaml
inspectit:
  instrumentation:
    rules:
      servlet_api_service:
        tracing:
          start-span: true
          sample-probability: 0.2
```

The example shown above will ensure that only 20% of all traces starting at the given rule will actually be exported.
Instead of specifying a fixed value, you can also specify a data key here, just like for `name`.
In this case, the value from the given data key is read and used as sampling probability.
This allows you for example to vary the sample probability based on the HTTP url.

If no sample probability is defined for a rule, the [default probability](tracing/tracing.md) is used.

Another useful property of spans is that you can attach any additional information in form of attributes.
In most tracing backends such as ZipKin and Jaeger, you can search your traces based on attributes.
The example below shows how you can define attributes:

```yaml
inspectit:
  instrumentation:
    rules:
      servlet_api_service:
        tracing:
          attributes:
            http_host: host_name
        entry:
          host_name:
           #... action call to fetch the http host here
```

The attributes property maps the names of attributes to data keys.
After the rule's exit phase, the corresponding data keys are read and attached as attributes to the current span.

Note that a rule does not have to start a span for attatching attributes.
If a rule does not start a span, the attributes will be written to the first span opened by any method on the current call stack.

It is also possible to conditionalize the span starting as well as the attribute writing:

```yaml
inspectit:
  instrumentation:
    rules:
      span_starting_rule:
        tracing:
          start-span: true
          start-span-conditions:
            only-if-true: my_condition_data
#....
      attribute_writing_rule:
        tracing:
          attributes:
            attrA: data_a
            attrB: data_b
          attribute-conditions:
            only-if-true: my_condition_data
```

If any `start-span-conditions` are defined, a span will only be created when all conditions are met.
Analogous to this, attributes will only be written if each condition defined in `attribute-conditions` is fulfilled.
The conditions that can be defined are equal to the ones of actions, thus, please see the [action conditions description](#adding-conditions) for detailed information.

With the previous shown settings, it is possible to add an instrumentation which creates exactly one span per invocation of an instrumented method. Especially in asynchronous scenarios, this might not be the desired behaviour:
For these cases inspectIT Ocelot offers the possibility to record multiple method invocations into a single span.
The resulting span then has the following properties:

* the span starts with the first invoked method and ends as soon as the last one returns
* all attributes written by each method are combined into the single span
* all invocations made from the methods which compose the single span will appear as children of this span

This can be configured by defining for rules that they (a) can continue existing spans and (b) can optionally end the span they started or continued.

Firstly, it is possible to "remember" the span created or continued using the `store-span` option:

```yaml
    rules:
      span_starting_rule:
        tracing:
          start-span: true
          store-span: my_span_data
          end-span: false
```

With this option, the span created at the end of the entry phase will be stored in the context with the data key `my_span_data`. Usually this span reference is then extracted from the context and attached ot an object via the [_attachments](#input-parameters).

Without the `end-span: false` definition above, the span would be ended as soon as the instrumented method returns.
By setting `end-span` to false, the span is kept open instead. It can then be continued when another method is executed as follows:

```yaml
    rules:
      span_finishing_rule:
        tracing:
          continue-span: my_span_data
          end-span: true # actually not necessary as it is the default value
```

Methods instrumented with this rule will not create a new span. Instead at the end of the entry phase the data for the key `my_span_data` is read from the context. If it contains a valid span written via `store-span`, this span is continued in this method. This implies that all spans started by callees of this method will appear as children of the span stored in `my_span_data`. In addition, this rule also then causes the continued span to end with the execution of the method due to the `end-span` option. This is not required to happen: a span can be continued by any number of rules before it is finally ended.

It also is possible to define rules for which both `start-span` and `continue-span` is configured.
In this case, the rule will first attempt to continue the existing span. Only if this fails (e.g. because the specified span does not exist yet or the conditions are not met) a new span is started.

Again, conditions for the span continuing and span ending can be specified just like for the span starting.
The properties `continue-span-conditions` and `end-span-conditions` work just like `start-span-conditions`.

