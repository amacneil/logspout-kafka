# logspout-kafka

A [Logspout](https://github.com/gliderlabs/logspout) adapter for writing Docker container logs to [Kafka](https://github.com/apache/kafka) topics. This fork has been modified from [gettyimages/logspout-kafka](https://github.com/gettyimages/logspout-kafka) to automatically format messages as JSON, and add the container id/name/image/hostname, similar to [logspout-logstash](https://github.com/looplab/logspout-logstash).

## Usage

To use this Kafka adapter, you must create a custom logspout docker image. The adapter is written in Go, but you do not need Go installed on your local computer to build it. First, create the following two files in a new directory:

**`Dockerfile`**
```Dockerfile
FROM gliderlabs/logspout:master
```

**`modules.go`**
```go
package main

import (
    _ "github.com/amacneil/logspout-kafka"
)
```

Next, build and tag the Docker image:

```sh
$ docker build -t mycompany/logspout .
```

Finally, you can run the image. The simplest case (sending all logs from Docker to Kafka) requires minimal setup. You must pass the `ROUTE_URIS` environment variable to specify the location of your Kafka broker server:

```sh
$ docker run --rm \
    --name="logspout" \
    -e LOGSPOUT=ignore \
    -e ROUTE_URIS="kafka://kafka-broker:9092/?topic=logstash" \
    --volume=/var/run/docker.sock:/var/run/docker.sock \
    mycompany/logspout
```

Note that the Kafka topic is specified by appending `?topic=foo` to the URI.

More info about building custom modules is available at the **logspout** project: [Custom Logspout Modules](https://github.com/gliderlabs/logspout/blob/master/custom/README.md)

## Configuration

The following environment variables are available to customize behavior of this container:

**`DEBUG=1`**

Enable debug logging.

**`KAFKA_CONNECT_RETRIES=1`**

Specify the number of times the Kafka connection will be retried (default 3).

**`KAFKA_COMPRESSION_CODEC=snappy`**

Set the Kafka compression to either `gzip` or `snappy` (default is no compression).

**`KAFKA_TEMPLATE="time=\"{{.Time}}\" container_name=\"{{.Container.Name}}\" source=\"{{.Source}}\" data=\"{{.Data}}\""`**

Specify a template string for the messages, instead of the default JSON formatting.

## Route Configuration (optional)

If you've mounted a volume to `/mnt/routes`, then consider pre-populating your routes. The following script configures a route to send standard messages from a "cat" container to one Kafka topic, and a route to send standard/error messages from a "dog" container to another topic.

```
cat > /logspout/routes/cat.json <<CAT
{
  "id": "cat",
  "adapter": "kafka",
  "filter_name": "cat_*",
  "filter_sources": ["stdout"],
  "address": "kafka-broker1:9092,kafka-broker2:9092/cat-logs"
}
CAT

cat > /logspout/routes/dog.json <<DOG
{
  "id": "dog",
  "adapter": "kafka",
  "filter_name": "dog_*",
  "filter_sources": ["stdout", "stderr"],
  "address": "kafka-broker1:9092,kafka-broker2:9092/dog-logs"
}
DOG

docker run --name logspout \
  -p "8000:8000" \
  --volume /logspout/routes:/mnt/routes \
  --volume /var/run/docker.sock:/tmp/docker.sock \
  gettyimages/example-logspout

```

The routes can be updated on a running container by using the **logspout** [Route API](https://github.com/gliderlabs/logspout/tree/master/routesapi) and specifying the route `id` "cat" or "dog".
