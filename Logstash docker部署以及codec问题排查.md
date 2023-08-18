# Logstash docker部署以及codec问题排查

LogStash常见于ELK stack，但是LogStash也可以脱离于elasticsearch独立使用。

当前问题为ElasticSearch出现大量错误日志堆积，通过排查后发现当前LogStash使用codec插件对接受到的JSON数据进行解析，但是数据源可能传输非JSON数据，再排查codec插件发现codec json插件源码默认接收到的数据为JSON格式，解析非JSON数据时直接报错。

如下使用docker对该错误进行复现，因为docker环境隔离，排除环境的问题。

1. 拉取镜像

```jsx
docker pull docker.elastic.co/logstash/logstash:8.9.0
```

1. 在/tmp目录下新建文件logstash.yml，events.txt以及pipeline文件夹，在pipeline文件夹中新建logstash.conf文件

logstash.yml如下，该配置源关闭ElasticSearch输出

```jsx
xpack.monitoring.enabled: false
```

events.txt如下，该文件用来测试JSON解析正确性

```jsx
{"id":1,"something":"text1"}
{"id":23,"something":"text1"}
fafgag
"id":23,"something":"text1"}
{:23,"something":"text1"}
{"id":23,"something":"text1"
```

logstash.conf文件如下，

```jsx
input {
    file {
        path => ["/tmp/events.txt"]
        start_position => "beginning"
        sincedb_path=> "/dev/null"
				codec => "json"
        add_field => {
            "file_path" => "/home"
        }
    }
}
output {
    stdout { codec => rubydebug { metadata => true } }
}
```

1. 在命令行输入命令

```jsx
docker run --name logstash  --rm -it -v /tmp/pipeline/:/usr/share/logstash/pipeline/ -v /tmp/logstash.yml:/usr/share/logstash/config/logstash.yml -v /tmp/events.txt:/tmp/events.txt docker.elastic.co/logstash/logstash:8.9.0
```

即可复现环境的报错

问题解决

查看logstash插件源码以及论坛，可以看到codec对非JSON数据日志报ERROR错误，使用JSON插件对非JSON数据解析报WARN。

同时可以发现json插件的skip_on_invalid_json和tag_on_failure配置选项逻辑有一定的问题

解决方式有如下

1. 重写logstash插件，并在生产环境配置，配置较为复杂
2. 根据JSON日志格式，使用gork正则将JSON数据过滤，但是每次都会调用正则，消耗一定的cpu性能
3. 使用filter的ruby code插件

最后选择方法3，logstash.conf文件如下

```jsx
input {
    file {
        path => ["/home/events.txt"]
        start_position => "beginning"
        sincedb_path=> "/dev/null"
        add_field => {
            "file_path" => "/home"
        }
    }
}

filter {
    ruby {
        init => "require 'json'"
        code => "
            begin
                parsed = JSON.parse(event.get('message'))
            rescue => e
                # event.tag('_jsoncheckfailure')
                event.cancel
            end
        "
    }

    json {
        source => "message"
        target => "message"
    }
}

output {
    stdout { codec => rubydebug { metadata => true } }
}
```

ruby code插件和json插件有一定的逻辑重合，会消耗性能，但是不想写了~~~