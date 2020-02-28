 apigateway
=============
A performant API Gateway based on Openresty and NGINX.

Table of Contents
=================

* [Status](#status)
* [Quick start](#quick-start)
* [What's inside](#whats-inside)
* [Performance](#performance)
* [Developer Guide](#developer-guide)

Status
======

The current project is considered production ready.


Quick start
===========

## Standalone

```bash
$ docker run --name="apigateway" \
            -p 80:80 \
            -e "LOG_LEVEL=info" \
            adobeapiplatform/apigateway:latest
```

## With Apache Mesos

```bash
$ docker run --name="apigateway" \
            -p 80:80 \
            -e "MARATHON_HOST=http://<marathon_host>:<port>" \
            -e "LOG_LEVEL=info" \
            adobeapiplatform/apigateway:latest
```

This command starts an API Gateway that automatically discovers the services running in Marathon.
The discovered services are exposed on individual VHosts as you can see in the [config file](https://github.com/adobe-apiplatform/apigateway/blob/master/api-gateway-config/conf.d/marathon_apis.conf#L36).

#### Accessing a Marathon app locally

For example, if you have an application named `hello-world` you can access it on its VHost in 2 ways:

 1. Edit `/etc/hosts` and add `<docker_host_ip> hello-world.api.localhost` then browse to `http://hello-world.api.localhost`
 2. Sending the Host header in a curl command: `curl -H "Host:hello-world.api.localhost" http://<docker_host_ip>`

The [discovery script](https://github.com/adobe-apiplatform/apigateway/blob/master/api-gateway-config/marathon-service-discovery.sh) is provided as an example for a quick-start and it can be replaced with your favourite discovery mechanism.
The script updates a [configuration file](https://github.com/adobe-apiplatform/apigateway/blob/master/api-gateway-config/environment.conf.d/api-gateway-upstreams.http.conf) containing all the NGINX upstreams that are used in the [config file](https://github.com/adobe-apiplatform/apigateway/blob/master/api-gateway-config/conf.d/marathon_apis.conf#L36).

#### Accessing a Marathon app remotely

All you need to do is to create a DNS entry like `*.api.example.com` or `*.gw.example.com` and have it resolve to the nodes running the Gateway.

Assuming there is an application deployed in Marathon called `hello-world` it can be accessed at the URL: `http://hello-world.api.example.com`
The Gateway is automatically proxying the request to the `hello-world` application in Marathon.

If you call `http://my-custom-app.api.example.com` the Gateway will proxy to a Marathon app named `my-custom-app`.
If the application is not deployed in Marathon a 5xx error is returned back to the client. This behaviour is ofcourse configurable.

#### Running with a different configuration folder

One way to simplify the configuration of the Gateway is to copy the `api-gateway-config` folder in S3, having all nodes syncing from that location.
For the moment only `AWS S3` is supported but the plan is to add support for other clouds too.

In order to work with S3 the Gateway needs to know the bucket. The command is similar to the one above with an extra environment variable named `REMOTE_CONFIG`
```
docker run --name="apigateway" \
            -p 80:80 \
            -e "MARATHON_HOST=http://<marathon_host>:<port>" \
            -e "REMOTE_CONFIG=s3://api-gateway-config-bucket" \
            -e "LOG_LEVEL=info" \
            adobeapiplatform/apigateway:latest
```

`s3://api-gateway-config-bucket` should contain something similar to what's in the [api-gateway-config](https://github.com/adobe-apiplatform/apigateway/blob/master/api-gateway-config/) folder.
For any customization it would be good to start from there.
There are only 2 files that won't be synchronised:
* `/etc/api-gateway/conf.d/includes/resolvers.conf` . Read bellow for more info
* `https://github.com/adobe-apiplatform/apigateway/blob/master/api-gateway-config/environment.conf.d/api-gateway-upstreams.http.conf` which is automatically generated by the Marathon Discovery script.

By default the access to the S3 bucket is done using IAM roles, but AWS Credentials can also be specified through environment variables:
```
docker run --name="apigateway" \
            -p 80:80 \
            -e "MARATHON_HOST=http://<marathon_host>:<port>" \
            -e "REMOTE_CONFIG=s3://api-gateway-config-bucket" \
            -e "AWS_ACCESS_KEY_ID=--change-me--" \
            -e "AWS_SECRET_ACCESS_KEY=--change-me--" \
            -e "LOG_LEVEL=info" \
            adobeapiplatform/apigateway:latest
```
By default the remote config is checked for changes every `10s`. This interval is configurable through the `REMOTE_CONFIG_SYNC_INTERVAL` env variable.
There are a few things to take into consideration when changing this value:
* How often the configs really change
* Cost. Some tools may make 1 API request per file to compare it. I.e. in S3 72 config files checked every `10s` costs `$7.46` but when checked every `30s` it's only `$2.4`, times number of GW nodes.
* Average time for an API Request. When reloading the GW the existing NGINX processes handling active connections are kept in the background until the request completes. So reloading the Gateway too fast may have the side effect of keeping too many processes running at the same time. This may, or may not be a problem but it's good to be aware of it.

#### Customizing the sync command

The sync command used for downloading the configuration files can be controlled via `REMOTE_CONFIG_SYNC_CMD` as well. This ENV VAR overrides the `REMOTE_CONFIG` one.

#### Resolvers
While starting up this container automatically creates the `/etc/api-gateway/conf.d/includes/resolvers.conf` config file using `/etc/resolv.conf` as the source.
To learn more about the `resolver` directive in NGINX see the [docs](http://nginx.org/en/docs/http/ngx_http_core_module.html#resolver).

#### Running the API Gateway outside of Marathon and Mesos
Besides the discovery part which is dependent on Marathon at the  moment, the API Gateway can run on its own as well. The Marathon service discovery is activated with the ` -e "MARATHON_HOST=http://<marathon_host>:<port>/"`.

#### Monitoring
Prometheus metrics are exposed on port `9113`. 

What's inside
=============


|   Module     |   Version  |  Details     |
|--------------|------------|--------------|
| [Openresty](https://github.com/openresty/) | 1.13.6.1 | Installed in `/usr/local/sbin/api-gateway` |
| [Openresty](https://github.com/openresty/) compiled `--with-debug` | 1.13.6.1 |  Installed in `/usr/local/sbin/api-gateway-debug` which enables [debugging log](http://nginx.org/en/docs/debugging_log.html) |
| [Test Nginx](https://github.com/openresty/test-nginx) | [0.24](https://github.com/openresty/test-nginx/releases/tag/v0.24) | Useful for executing integration tests from the container. <br/> It's installed in `/usr/local/test-nginx-0.24/`. <br/> It's also used during Docker build to execute `make test` on lua modules.  |
| [PCRE](https://sourceforge.net/projects/pcre/) | [8.37](https://sourceforge.net/projects/pcre/files/pcre/8.37/) | Enables PCRE JIT support | 
| [ZeroMQ](http://download.zeromq.org/) | [4.0.5](http://zeromq.org/area:download) |  ZeroMQ |
| [CZMQ](http://download.zeromq.org/) | [2.2.0](http://czmq.zeromq.org/page:get-the-software) |  CZMQ - High-level C Binding for ZeroMQ |


##### Modules for API Management and Logging

|   Module     |   Version  |  Description |
|--------------|------------|--------------|
| [api-gateway-config-supervisor](https://github.com/adobe-apiplatform/api-gateway-config-supervisor) | [1.0.3](https://github.com/adobe-apiplatform/api-gateway-config-supervisor/releases/tag/1.0.3) | Syncs config files from Amazon S3 reloading the gateway with the updates |
| [api-gateway-cachemanager](https://github.com/adobe-apiplatform/api-gateway-cachemanager) | [1.0.1](https://github.com/adobe-apiplatform/api-gateway-cachemanager/releases/tag/1.0.1) | Lua library for managing multiple cache stores |
| [api-gateway-hmac](https://github.com/adobe-apiplatform/api-gateway-hmac) | [1.0.0](https://github.com/adobe-apiplatform/api-gateway-hmac/releases/tag/1.0.0) | HMAC support for Lua with multiple algorithms, via OpenSSL and FFI |
| [api-gateway-aws](https://github.com/adobe-apiplatform/api-gateway-aws) | [1.7.1](https://github.com/adobe-apiplatform/api-gateway-aws/releases/tag/1.7.1) | AWS SDK for Nginx in Lua |
| [api-gateway-request-validation](https://github.com/adobe-apiplatform/api-gateway-request-validation) | [1.2.4](https://github.com/adobe-apiplatform/api-gateway-request-validation/releases/tag/1.2.4) | API Request Validation framework  |
| [api-gateway-async-logger](https://github.com/adobe-apiplatform/api-gateway-async-logger) | [1.0.1](https://github.com/adobe-apiplatform/api-gateway-async-logger/releases/tag/1.0.1) | Performant async logger |
| [api-gateway-zmq-logger](https://github.com/adobe-apiplatform/api-gateway-zmq-logger) | [1.0.0](https://github.com/adobe-apiplatform/api-gateway-zmq-logger/releases/tag/1.0.0) | Lua logger for ZMQ with FFI and CZMQ |
| [api-gateway-request-tracking](https://github.com/adobe-apiplatform/api-gateway-request-tracking) | [1.0.1](https://github.com/adobe-apiplatform/api-gateway-request-tracking/releases/tag/1.0.1) | Usage and Tracking Handler for the API Gateway |

##### Other Lua Modules

|   Module     |   Version  |  Description |
|--------------|------------|--------------|
| [lua-resty-http](https://github.com/pintsized/lua-resty-http) | [v0.07](https://github.com/pintsized/lua-resty-http/releases/tag/v0.07) | Lua HTTP client cosocket driver for OpenResty / ngx_lua |
| [lua-resty-iputils](https://github.com/hamishforbes/lua-resty-iputils) | [v0.2.0](https://github.com/hamishforbes/lua-resty-iputils/releases/tag/v0.2.0) | Utility functions for working with IP addresses in Openresty |

Performance
===========

The following performance tests results have been obtained on a virtual machine with 8 CPU cores and 4GB Memory.

The API Gateway container has been started with 4 CPU cores and `net=host` :
```bash
docker run --cpuset-cpus=0-3 --net=host --name="apigateway" -e "LOG_LEVEL=notice" adobeapiplatform/apigateway:latest
```

WRK test has been started with 4 CPU cores and `net=host`:
```bash
docker run --cpuset-cpus=4-7 --net=host williamyeh/wrk:4.0.1 -t4 -c1000 -d30s http://<docker_host_ip>/health-check
Running 30s test @ http://192.168.75.158/health-check
  4 threads and 1000 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency    30.38ms   73.80ms   1.16s    90.90%
    Req/Sec    35.26k    11.98k   83.70k    68.72%
  4214013 requests in 30.06s, 1.28GB read

Requests/sec: 140165.09

Transfer/sec:     43.57MB
```

Developer guide
================

 To build the docker image locally use:
 ```
  make docker
 ```

 To SSH into the newly built image use ( note that this is not the running image):
 ```
  make docker-ssh
 ```

#### Running and Stopping the Docker image
 ```
  make docker-run
 ```
 The main API Gateway process is exposed to port `80`. To test that the Gateway works see its `health-check`:
 ```
  $ curl http://<docker_host_ip>/health-check
    API-Platform is running!
 ```
 If you're up for a quick performance test, you can play with Apache Benchmark via Docker:

 ```
  docker run jordi/ab ab -k -n 200000 -c 500 http://<docker_host_ip>/health-check
 ```

 To run docker mounting the local `api-gateway-config` directory into `/etc/api-gateway/` issue:

 ```bash
 $ make docker-debug
 ```
 In debug mode the docker container starts a special `api-gateway` compiled `--with-debug` providing very detailed debugging information.
When started with `-e "LOG_LEVEL=info"` the output is quite verbose.
To learn more about this option visit [NGINX docs](http://nginx.org/en/docs/debugging_log.html).

 When done stop the image:
 ```
 make docker-stop
 ```

### Enabling API KEY Management through Redis

This command starts two docker containers: redis and gateway
 ```
 make docker-compose
 ```

### SSH into the running image

```
make docker-attach
```

#### Running the container in Mesos with Marathon

Make an HTTP POST on `http://<marathon-host>/v2/apps` with the following payload.
For optimal performance leave the `network` on `HOST` mode. To learn more about the network modes visit the Docker [documentation](https://docs.docker.com/articles/networking/#how-docker-networks-a-container).

```javascript
{
  "id": "api-gateway",
  "container": {
    "type": "DOCKER",
    "docker": {
      "image": "adobeapiplatform/apigateway:latest",
      "forcePullImage": true,
      "network": "HOST"
    }
  },
  "cpus": 4,
  "mem": 4096.0,
  "env": {
    "MARATHON_HOST": "http://<marathon_host>:<marathon_port>"
  },
  "constraints": [  [ "hostname","UNIQUE" ]  ],
  "ports": [ 80 ],
  "healthChecks": [
    {
      "protocol": "HTTP",
      "portIndex": 0,
      "path": "/health-check",
      "gracePeriodSeconds": 3,
      "intervalSeconds": 10,
      "timeoutSeconds": 10
    }
  ],
  "instances": 1
}
```

To run the Gateway only on specific nodes marked with `slave_public` you can add the property bellow to the main JSON object:
```
"acceptedResourceRoles": [ "slave_public" ]
```

##### Auto-discover and register Marathon tasks in the Gateway

To enable auto-discovery in a Mesos with Marathon framework define the following Environment Variables:
```
MARATHON_URL=http://<marathon-url-1>
MARATHON_TASKS=ws-.* ( NOT USED NOW. TBD IF THERE'S A NEED TO FILTER OUT SOME TASKS )
```

So the Docker command is now :
```
docker run --name="apigateway" \
            -p 8080:80 \
            -e "MARATHON_HOST=http://<marathon_host>:<port>/" \
            -e "LOG_LEVEL=info" \
            adobeapiplatform/apigateway:latest
```

