# Docker image for logstash with x-pack and preset pipeline conf

## Env variables
To work the image needs the following env variables:
- TCP_PORT: it's the port that logstash will use to listen for new connections
- ELASTICSEARCH_HOST: The hostname for the elasticsearch service
- ELASTICSEARCH_PORT: The port for the elasticsearch service
