
## Description

A full ELK stack, with x-pack plugin installed, to be run on docker swarm or with docker-compose.

## X-pack 

Note: this info is true at the date of this commit, it might change in the future in a unforseeable way for me so always refer to elastic website for accurate and up to date informations.

X-Pack is a mostly paid plugin, it gives you authentication for the full stack along side with other useful tools. It has a free trial of 30 days and then will revert to a free version where authentication is disabled, yet the monitoring extras are quite good.

## Log with logback appender

```xml
<appender name="LOGSTASH" class="net.logstash.logback.appender.LogstashTcpSocketAppender">
    <keepAliveDuration>5 minutes</keepAliveDuration>
    <reconnectionDelay>10 second</reconnectionDelay>
    <waitStrategyType>sleeping</waitStrategyType>
    <ringBufferSize>16384</ringBufferSize>
    <destination>127.0.0.1:5000</destination>

    <encoder class="net.logstash.logback.encoder.LoggingEventCompositeJsonEncoder">
        <providers>
            <mdc/>
            <context/>

            <logLevel/>
            <loggerName/>
            <threadName />

            <pattern>
                <pattern>
                    {
                        "context": "rco.backoffice"
                    }
                </pattern>
            </pattern>

            <message/>

            <logstashMarkers/>
            <arguments/>

            <stackTrace>
                <throwableConverter class="net.logstash.logback.stacktrace.ShortenedThrowableConverter">
                    <exclude>net\.sf\.cglib\..*</exclude>
                    <maxDepthPerThrowable>30</maxDepthPerThrowable>
                    <rootCauseFirst>true</rootCauseFirst>
                </throwableConverter>
            </stackTrace>
        </providers>
    </encoder>
</appender>
```

## Test/configuration commands

The following commands could help you getting started with smoke test of Elasticsearch, a first import of log in logstash and the configuration of kibana for logstash logs

#-u elastic:changeme

### Elasticsearch

curl -XGET 'http://localhost:9200'

### Logstash

nc localhost 5000 < *.log

### Kibana

curl -XPOST -D- 'http://localhost:5601/api/saved_objects/index-pattern' \
    -H 'Content-Type: application/json' \
    -H 'kbn-version: 6.2.3' \
    -d '{"attributes":{"title":"logstash-*","timeFieldName":"@timestamp"}}'

## Manually run services

### Portainer - visual ui for docker

```
docker service create \
    --name portainer \
    --publish 9999:9000 \
    --replicas=1 \
    --constraint 'node.role == manager' \
    --mount type=bind,src=//var/run/docker.sock,dst=/var/run/docker.sock \
    --mount src=portainer_data,dst=/data \
    portainer/portainer \
    -H unix:///var/run/docker.sock
```

### Elasticsearch

Using a docker volume:

```
docker service create \
    --name elasticsearch \
    --publish 9200:9200 \
    --publish 9300:9300 \
    --replicas=1 \
    --env CLUSTER_NAME="elk-cluster" \
    --env ELASTIC_PASSWORD=changeme \
    --env NETWORK_HOST="0.0.0.0" \
    --env DISCOVERY_ZEN_MINIMUM_MASTER_NODES=1 \
    --env ES_JAVA_OPTS="-Xmx256m -Xms256m" \
    --env DISCOVERY_TYPE="single-node" \
    --mount src=elasticsearch_data,dst=/usr/share/elasticsearch/data \
    --network elk \
    docker.elastic.co/elasticsearch/elasticsearch-basic:6.2.3
```

You can use a filesystem bind instead of a docker volume, in this case replace the mount line with the following one changing 'path/to/data' with the appropriate absolute path:

```
    --mount type=bind,src=//path/to/data,dst=/usr/share/elasticsearch/data \
```

You can pass a configuration file instead of env variables (which are, in my opinion, more cloud friendly). If you choose to do so add a mount like this one:

```
    --mount type=bind,src=//Users/alessandro/projects/_learning/docker-elk/elasticsearch/config/elasticsearch.yml,dst=/usr/share/elasticsearch/config/elasticsearch.yml \
```

### Kibana

```
docker service create \
    --name kibana \
    --publish 5601:5601 \
    --replicas=1 \
    --env SERVER_NAME=kibana \
    --env SERVER_HOST="0.0.0.0" \
    --env ELASTICSEARCH_URL="http://elasticsearch:9200" \
    --env XPACK_MONITORING_UI_CONTAINER_ELASTICSEARCH_ENABLED=true \
    --env ELASTICSEARCH_USERNAME=elastic \
    --env ELASTICSEARCH_PASSWORD=changeme \
    --network elk \
    docker.elastic.co/kibana/kibana-x-pack:6.2.3
```

You can pass a configuration file instead of env variables (which are, in my opinion, more cloud friendly). If you choose to do so add a mount like this one:

```
    --mount type=bind,src=//Users/alessandro/projects/_learning/docker-elk/kibana/config/kibana.yml,dst=/usr/share/kibana/config/kibana.yml \
```

### Logstash 

Using the custom docker image build from the logstash directory, here we assume is published on dockerhub, you can use it from local or publish wherever it's more appropriate for your use case.

```
docker service create \
    --name logstash \
    --publish 5000:5000 \
    --replicas=1 \
    --env LS_JAVA_OPTS="-Xmx256m -Xms256m" \
    --env HTTP_HOST="0.0.0.0" \
    --env TCP_PORT="5000" \
    --env PATH_CONFIG=/usr/share/logstash/pipeline \
    --env ELASTICSEARCH_HOST=elasticsearch \
    --env ELASTICSEARCH_PORT=9200 \
    --env ELASTICSEARCH_USERNAME=elastic \
    --env ELASTICSEARCH_PASSWORD=changeme \
    --env XPACK_MONITORING_ELASTICSEARCH_URL="http://elasticsearch:9200" \
    --env XPACK_MONITORING_ELASTICSEARCH_USERNAME=logstash_system \
    --env XPACK_MONITORING_ELASTICSEARCH_PASSWORD=changeme \
    --network elk \
    northernlightsio/elk_logstash-x-pack:6.2.3
```

Usage with official docker image and external files for configuration and pipeline definition:

```
    --mount type=bind,src=//Users/alessandro/projects/_learning/docker-elk/logstash/config/logstash.yml,dst=/usr/share/logstash/config/logstash.yml \
    --mount type=bind,src=//Users/alessandro/projects/_learning/docker-elk/logstash/pipeline,dst=/usr/share/logstash/pipeline \
    docker.elastic.co/logstash/logstash-x-pack:6.2.3
```


