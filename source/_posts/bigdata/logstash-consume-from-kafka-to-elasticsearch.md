---
date: 2018-03-20 14:29
categories: 大数据
tags: 
	-kafka
	-logstash
title: 利用logstash从kafka消费数据到elasticsearch

---



##  1. 几种方式

	目前要把kafka中的数据传输到elasticsearch集群大概有一下几种方法：

 -  logstash

- flume

- spark streaming

- kafka connect

- 自己开发程序读取kafka写入elastic

  

          其中logstash看到网上说不太稳定，且我目前用过版本2.3 ，确实经常出现crash的情况，所以一开始并未考虑；首先尝试的是通过flume到es，因为目前kafka到HDFS中间用的是flume，想再加一个通道和sink到es，而且flume也有es的sink，但是我的flume是最新版1.8，elasticsearch也是最新版6.2.2，中间碰到了兼容性问题，未能成功；转而去研究kafka connect，按照《kafka权威指南》上的例子研究了一下，同样遇到兼容性问题，在我的版本组合中无法奏效，但我不想去修改已经安装好的flume或者es集群，spark streaming过于复杂，自己开发程序成本过高、且周期较长；最终去尝试logstash，结果配置非常容易，简单奏效，稳定性问题暂时无法看出，留待日后详测，现记录一下配置。

## 2. logstash配置

### 2.1 kafka input plugin

```
input{
      kafka{
        bootstrap_servers => ["192.168.1.120:9092,192.168.1.121:9092,192.168.1.122:9092"]
        client_id => "test"
        group_id => "logstash-es"
        auto_offset_reset => "latest" 
        consumer_threads => 5
        decorate_events => "true"
        topics => ["auth","base","clearing","trademgt"] 
        type => "kafka-to-elas"
      }
}
```

其中配置项**decorate_events** 颇为有用，如果只用了单个logstash，订阅了多个主题，你肯定希望在es中为不同主题创建不同的索引，那么**decorate_events** 就是你想要的，看官方解释：

#### `decorate_events`

- Value type is  boolean
- Default value is `false`

Option to add Kafka metadata like topic, message size to the event. This will add a field named `kafka` to the logstash event containing the following attributes: `topic`: The topic this message is associated with `consumer_group`: The consumer group used to read in this event `partition`: The partition this message is associated with `offset`: The offset from the partition this message is associated with `key`: A ByteBuffer containing the message key

大意是指定这个选项为**true**时，会附加kafka的一些信息到logstash event的一个名为`kafka`的域中，例如topic、消息大小、偏移量、consumer_group等，具体如下：

### Metadata fields

The following metadata from Kafka broker are added under the `[@metadata]` field:

- `[@metadata][kafka][topic]`: Original Kafka topic from where the message was consumed.
- `[@metadata][kafka][consumer_group]`: Consumer group
- `[@metadata][kafka][partition]`: Partition info for this message.
- `[@metadata][kafka][offset]`: Original record offset for this message.
- `[@metadata][kafka][key]`: Record key, if any.
- `[@metadata][kafka][timestamp]`: Timestamp when this message was received by the Kafka broker.

Please note that `@metadata` fields are not part of any of your events at output time. If you need these information to be inserted into your original event, you’ll have to use the `mutate` filter to manually copy the required fields into your `event`.

> 值得注意的是，在output的时候这些域的元数据信息并不是event的一部分，如果希望这些元数据插入到原生的event中，就需要利用`mutate`手动copy进event，我们接下来会在filter中利用`kafka`域的内容构建自定义的域。

### 2.2  构建`[@metadata][index]`

```
filter {
   if [@metadata][kafka][topic] == "auth" {
      mutate {
         add_field => {"[@metadata][index]" => "auth-%{+YYYY.MM.dd}"}
      }
   } else if [@metadata][kafka][topic] == "base" {
      mutate {
         add_field => {"[@metadata][index]" => "base-%{+YYYY.MM.dd}"}
      }

   }else if [@metadata][kafka][topic] == "clearing" {

      mutate {
         add_field => {"[@metadata][index]" => "clearing-%{+YYYY.MM.dd}"}
      }
   }else{
      mutate {
         add_field => {"[@metadata][index]" => "trademgt-%{+YYYY.MM.dd}"}
      }
   }
   # remove the field containing the decorations, unless you want them to land into ES
   mutate {
      remove_field => ["kafka"]
   }
}
```



为每个主题构建对应的`[@metadata][index]`，并在接下来output中引用

### 2.3 elasticsearch output plugin

```
output{
        elasticsearch{
               hosts => ["192.168.1.123:9200","192.168.1.122:9200","192.168.1.121:9200"]
               index => "%{[@metadata][index]}"
               timeout => 300
          }

}
```

启动`logstash`就可以在`kibana`中观察到数据了。

## 3. 特殊的metadata

**The @metadata field**

In Logstash 1.5 and later, there is a special field called `@metadata`. The contents of `@metadata` will not be part of any of your events at output time, which makes it great to use for conditionals, or extending and building event fields with field reference and sprintf formatting.

The following configuration file will yield events from STDIN. Whatever is typed will become the `message` field in the event. The `mutate` events in the filter block will add a few fields, some nested in the `@metadata` field.

```
input { stdin { } }

filter {
  mutate { add_field => { "show" => "This data will be in the output" } }
  mutate { add_field => { "[@metadata][test]" => "Hello" } }
  mutate { add_field => { "[@metadata][no_show]" => "This data will not be in the output" } }
}

output {
  if [@metadata][test] == "Hello" {
    stdout { codec => rubydebug }
  }
}
```

Let’s see what comes out:

```
$ bin/logstash -f ../test.conf
Pipeline main started
asdf
{
    "@timestamp" => 2016-06-30T02:42:51.496Z,
      "@version" => "1",
          "host" => "example.com",
          "show" => "This data will be in the output",
       "message" => "asdf"
}
```

The "asdf" typed in became the `message` field contents, and the conditional successfully evaluated the contents of the `test` field nested within the `@metadata` field. But the output did not show a field called `@metadata`, or its contents.

The `rubydebug` codec allows you to reveal the contents of the `@metadata` field if you add a config flag, `metadata => true`:

```
    stdout { codec => rubydebug { metadata => true } }
```

Let’s see what the output looks like with this change:

```
$ bin/logstash -f ../test.conf
Pipeline main started
asdf
{
    "@timestamp" => 2016-06-30T02:46:48.565Z,
     "@metadata" => {
           "test" => "Hello",
        "no_show" => "This data will not be in the output"
    },
      "@version" => "1",
          "host" => "example.com",
          "show" => "This data will be in the output",
       "message" => "asdf"
}
```

Now you can see the `@metadata` field and its sub-fields.

![Important](https://www.elastic.co/guide/en/logstash/6.2/images/icons/important.png)

Only the `rubydebug` codec allows you to show the contents of the `@metadata` field.

Make use of the `@metadata` field any time you need a temporary field but do not want it to be in the final output.

Perhaps one of the most common use cases for this new field is with the `date` filter and having a temporary timestamp.

This configuration file has been simplified, but uses the timestamp format common to Apache and Nginx web servers. In the past, you’d have to delete the timestamp field yourself, after using it to overwrite the `@timestamp` field. With the `@metadata` field, this is no longer necessary:

```
input { stdin { } }

filter {
  grok { match => [ "message", "%{HTTPDATE:[@metadata][timestamp]}" ] }
  date { match => [ "[@metadata][timestamp]", "dd/MMM/yyyy:HH:mm:ss Z" ] }
}

output {
  stdout { codec => rubydebug }
}
```

Notice that this configuration puts the extracted date into the `[@metadata][timestamp]` field in the `grok` filter. Let’s feed this configuration a sample date string and see what comes out:

```
$ bin/logstash -f ../test.conf
Pipeline main started
02/Mar/2014:15:36:43 +0100
{
    "@timestamp" => 2014-03-02T14:36:43.000Z,
      "@version" => "1",
          "host" => "example.com",
       "message" => "02/Mar/2014:15:36:43 +0100"
}
```

That’s it! No extra fields in the output, and a cleaner config file because you do not have to delete a "timestamp" field after conversion in the `date` filter.

Another use case is the CouchDB Changes input plugin (See <https://github.com/logstash-plugins/logstash-input-couchdb_changes>). This plugin automatically captures CouchDB document field metadata into the `@metadata` field within the input plugin itself. When the events pass through to be indexed by Elasticsearch, the Elasticsearch output plugin allows you to specify the `action`(delete, update, insert, etc.) and the `document_id`, like this:

```
output {
  elasticsearch {
    action => "%{[@metadata][action]}"
    document_id => "%{[@metadata][_id]}"
    hosts => ["example.com"]
    index => "index_name"
    protocol => "http"
  }
}
```