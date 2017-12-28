# Distributed Logging Example (dle)
This is a Docker Compose project showcasing a distributed logging architecture. This example uses docker, nginx, fluentd, MongoDB, elasticsearch, and a local AWS S3 mock to simulate how a production setup might look like. Fluentd is at the heart of this solution and is used to capture, forward, and store logs in a variety of destinations.

## Containers
__dle-webapp1:__ nginx container serving simple static content, logs to the log agent container

__dle-log-agent:__ fluentd container listening for logs from containers on the same docker host

__dle-log-forwarder-pri:__ primary fluentd container receiving traffic from the log agent container and forwarding to several destinations

__dle-log-forwarder-sec:__ active backup fluentd forwarder container

__dle-log-store:__ mongodb container used for storing logs

__dle-log-search:__ elasticsearch container receiving and storing logs from fluentd forwarders

__dle-log-archive:__ local S3 mock container for archiving logs

## Running
```
git clone https://github.com/jlowsley/dle
cd dle
docker-compose up
```
_Note: the store, search, and archive containers take about a minute to initialize and become available_
## Testing
After the containers are initialized and running, and docker-compose logs come to a stop, the setup can be tested. Make a request to the web application container, using a browser or with curl:
```
$ curl localhost:8000
<html><p>Simple Static Content!</p></html>
```
or make several requests...
```
$ for i in {0..99}; do curl localhost:8000; done
```
After a few seconds logs will flow through the system and be available. Check to see that some entry has been made in all the expected locations.

__MongoDB (dle-log-store)__

Count the documents in the collection to see some entries.
```
$ docker exec -it dle-log-store mongo applogs --quiet --eval "db.docker.count()"
101
```
To view one of the documents:
```
$ docker exec -it dle-log-store mongo applogs --quiet --eval "db.docker.find().limit(1)"
...
```
__Elasticsearch (dle-log-search)__

Use the cat count API to see that the index now has some records.
```
$ curl "http://127.0.0.1:9200/_cat/count/logstash-`date -u +%Y.%m.%d`?v"
epoch      timestamp count
1514398977 18:22:57  101
```
To look at one of the entries:
```
$ curl "http://127.0.0.1:9200/logstash-`date -u +%Y.%m.%d`/_search?size=1&pretty=true"
...
```
__S3 (dle-log-archive)__

Listing the `applogs` bucket using the `logs` and date prefix should show an archived log file from the hour with a serial number.  
```
$ export AWS_ACCESS_KEY_ID=accessKey1
$ export AWS_SECRET_ACCESS_KEY=verySecretKey1
$ aws --endpoint-url http://localhost:9000 s3 ls s3://applogs/logs/`date -u +%Y-%m-%d`/
2017-12-27 11:21:22        475 18_0.gz
```
To copy it to the host to read it:
```
$ aws --endpoint-url http://localhost:9000 s3 cp s3://applogs/logs/`date -u +%Y-%m-%d`/18_0.gz .
download: s3://applogs/logs/2017-12-27/18_0.gz to ./18_0.gz
$ zcat 18_0.gz
...
```
__Simulate log forwarder failure__

To test fail over of the log forwarders:
```
$ docker-compose pause dle-log-forwarder-pri
```
After the timeout (5s) the log agent will then start using the secondary log forwarder container, and can be tested by making the same requests to the web application container. The primary can then be resumed:

```
$ docker-compose unpause dle-log-forwarder-pri
```

## Features
- several containers on a single docker host can leverage a single log agent
- the log agent is easily reconfigured to point to different forwarders by simply replacing hostnames in fluent.conf
- there are 2 forwarders for fault tolerance, the log agent will fail over to the secondary forwarder if the primary is unreachable
- log agent only sends matching docker logs to log forwarders, but can be reconfigured in fluent.conf
- buffering and flush intervals are set aggressively low for testing/troubleshooting, but can be changed in fluent.conf


## Use Cases
- good example of multiple container docker compose project
- can be used to develop fluentd configurations
- can be used to mock application logging for other local projects to feed into

## Improvements and Alternatives
- create additional example using CloudFormation for AWS native solution using ECS, Kinesis, RDS, DynamoDB, S3, AWS ES Service, CW Logs
- consider other stores like postgresql, CloudWatch Logs
- add another application container
- show different log types (nginx, apache, custom, etc.) using parsing, matching, and filters
- try logging straight to log-agent instead of through docker logging API
- try to mock AWS task/instance roles (keyless access) for S3 IAM permissions
