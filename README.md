# Example Configurations for Fluentd

## Inputs

### File Input

One of the most common types of log input is tailing a file. The in_tail input plugin allows you to read from a text log file as though you were running the _tail -f_ command. Full documentation on this plugin can be found [here](https://docs.fluentd.org/input/tail).

```
<source>
  @type tail
  <parse>
    @type none
  </parse>
  path /home/logs/*
  tag backend.application
</source>
```

In this tail example, we are declaring that the logs should not be parsed by seeting _@type none_. We are also adding a tag that will control routing. By setting _tag backend.application_ we can specify filter and match blocks that will only process the logs from this one source. More details on how routing works in Fluentd can be found [here](https://docs.fluentd.org/configuration/routing-examples). 

### Syslog Input

Another very common source of logs is syslog, This example will bind to all addresses and listen on the specified port for syslog messages.

```
<source>
  @type syslog
  port 5140
  tag syslog.messages
</source>
```

## Managing Data

### Adding Parsing

Sometimes you will have logs which you wish to parse. There is a set of built-in parsers listed [here](https://docs.fluentd.org/parser#list-of-built-in-parsers) which can be applied. Some of the parsers like the _nginx_ parser understand a common log format and can parse it "automatically." Others like the _regexp_ parser are used to declare custom parsing logic. There is also a very commonly used 3rd party parser for [grok](https://github.com/fluent/fluent-plugin-grok-parser) that provides a set of regex macros to simplify parsing.

This next example is showing how we could parse a standard NGINX log we get from file using the in_tail plugin. 

```
<source>
  @type tail
  <parse>
    @type nginx
  </parse>
  path /var/log/nginx/error.log
  tag nginx.error
</source>
```

Notice that we have chosen to tag these logs as _nginx.error_ to help route them to a specific output and filter plugin after. If we wanted to apply custom parsing the _grok_ filter would be an excellent way of doing it. In this next example, a series of _grok patterns_ are used. The first pattern is _%{SYSLOGTIMESTAMP:timestamp}_ which pulls out a timestamp assuming the standard syslog timestamp format is used. The next pattern grabs the log level and the final one grabs the remaining unnmatched txt. Each substring matched becomes an attribute in the log event stored in New Relic. This makes it possible to do more advanced monitoring and alerting later by using those attributes to filter, search and facet.

```
<source>
  @type tail
  <parse>
    @type grok
    <grok>
      pattern %{SYSLOGTIMESTAMP:timestamp} %{LOGLEVEL:loglevel}: %{GREEDYDATA:message}
    </grok>
  </parse>
  path /home/log/test.log
  tag custom.application
</source>
```

### Multiline Logs

Some logs have single entries which span multiple lines. Typically one log entry is the equivalent of one log line; but what if you have a stack trace or other long message which is made up of multiple lines but is logically all one piece? In that case you can use a multiline parser with a regex that indicates where to start a new log entry. A common start would be a timestamp; whenever the line begins with a timestamp treat that as the start of a new log entry. If the next line begins with something else, continue appending it to the previous log entry. 

```
 <filter backend.application>
    @type parser
    <parse>
      @type multiline_grok
      grok_failure_key grokfailure
      multiline_start_regex ^abc
      <grok>
        pattern %{GREEDYDATA:message}
      </grok>
    </parse>
  </filter>
```
The above example uses [multiline_grok](https://github.com/fluent/fluent-plugin-grok-parser#multiline-support) to parse the log line; another common parse filter would be the standard [multiline parser](https://docs.fluentd.org/parser/multiline). This is also the first example of using a [<filter>](https://docs.fluentd.org/filter). Multiple filters can be applied before matching and outputting the results. In the example, any line which begins with "abc" will be considered the start of a log entry; any line beginning with something else will be appended.
  
### Adding the service_name field

It is possible to add data to a log entry before shipping it. In Fluentd entries are called "fields" while in NRDB they are referred to as the attributes of an event. Different names in different systems for the same data. One important field for organizing your logs is the _service_name_ field. This is a reserved field name in New Relic along with _message_. 

```
<source>
  @type syslog
  port 5140
  tag backend.application
</source>

<filter backend.application>
  @type record_transformer
  <record>
    service_name ${tag}
  </record>
</filter>
```

This example makes use of the [record_transformer](https://docs.fluentd.org/filter/record_transformer) filter. It allows you to change the contents of the log entry (the record) as it passes through the pipeline. The field name is _service_name_ and the value is a variable _${tag}_ that references the tag value the filter matched on. The tag value of _backend.application_ set in the <source> block is picked up by the filter; that value is referenced by the variable. 

The result is that __"service_name: backend.application"__ is added to the record.

## Complete Examples

### Minimal Configuration

```
<source>
  @type tail
  <parse>
    @type none
  </parse>
  path /home/logs/*
  tag sample.tag
</source>

<filter sample.tag>
  @type record_transformer
  <record>
    service_name ${tag}
  </record>
</filter>

<match **>
  @type newrelic
  api_key <your key goes here>
</match>
```
