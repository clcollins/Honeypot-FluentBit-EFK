# Honeypot -> Fluentd/FluentBit -> EFK

Proof of concept architecture for forwarding honeypot data to an EFK (Elasticsearch, Fluentd, Kibana) stack with Fluentd or FluentBit, using Docker  containers.

## What are the things?

1.  docker-compose-efk.yml - EFK Stack POC
2.  docker-compose-fluent-bit.yml - Cowrie container logging to STDOUT, read by FluentBit container.
3.  docker-compose-fluentd.yml - Example Fluentd container
4.  .devel/ - Directory with associated Docker- and config-files (POC only - not production ready.)

## How would this be used?

1.  An EFK Stack is stood up to receive logs to the "F" - Fluentd, and store them in Elasticsearch.  Kibana is available for visualization.
2.  A Cowrie honeypot is stood up to log to STDOUT, which is read by FluentBit (or Fluentd) via the Docker `--log-driver fluentd` option.

Fluentd and FluentBit are configured to listen for TCP connections on port 24224.  Docker's `--log-driver fluentd` option can be configured globally, or, per-container, to forward traffic to a Fluentd instance - by default localhost:24224.

A local container running Fluentd or FluentBit is configured to accept incoming traffic, and forward to a remote Fluentd instance (the "F" in EFK), which receives the logs, and sends them to Elasticsearch.

This can be done with or without log filtering, transformation or de-duplication.

## What would I change?

**FluentBit output**

Currently, FluentBit with Cowrie logs to STDOUT.  This would be changed to forward to Fluentd in the EFK stack,
with or without TLS (TLS example shown below):

_Forward Output Plugin with TLS_

```
[OUTPUT]
    Name          forward
    Match         *
    Host          efk.host.addr
    Port          24284
    Shared_Key    secret
    Self_Hostname flb.local
    tls           on
    tls.verify    off
```

**EFK input**

The Fluentd instance in the EFK stack is configured in the examples to accept traffic from remote hosts on the standard Fluentd port.  If TLS is used (as in the example above), the input would be changed as well:

_Forward Input Plugin with TLS_

```
<source>
  @type         secure_forward
  self_hostname honey.pot.address
  shared_key    secret
  secure no
</source>
```

## Cool FluentBit option

FluentBit can accept configuration via the command line, so configuration files are not particularly necessary:

`bin/fluent-bit -i INPUT -o forward://HOST:PORT`

## Gotchas

**EFK Logs**

The EFK stack has commented-out sections for logging Elasticsearch and Kibana to Fluentd via the `--log-driver fluentd` Docker option in the `docker-compose-efk.yml` file.  These can be re-enabled, but make it harder to tell if Kibana or Elasticsearch are broken when first getting it setup.  You may prefer to leave these commented out until you have the EFK stack working.

**Elasticsearch & vm.max_map_count**

Elasticsearch needs a higher vm.max_map_count than is usually set in the Kernal of a system.  Run:

`sysctl -w vm.max_map_count=262144`

...to set to the lowest acceptable limit.

**Docker Compose STDOUT with --log-driver fluentd**

There is no STDOUT shown by Docker Compose if run in the foreground, while using the fluentd log driver.  You will receive warnings like:

`cowrie_1   | WARNING: no logs are available with the 'fluentd' log driver`

This is just a poorly worded message indicating the same as above.

**Startup order with --log-driver fluentd**

In order to use `--log-driver fluentd`, there must be a Fluentd/FluentBit instance listening wherever the log driver is configured to send logs when containers are started, or the containers will stop with an error message.

If your Docker Compose file starts up the local Fluentd/FluentBit instance that's used by containers in the same `docker-compose.yml`, those containers must use `depends_on:` to indicate they must start after the Fluentd/FluentBit container.

## Future consideration

Config management would be simplified (especially for secure forward source configuration) by employing a distributed key-value store / service discovery service.

## Compatibility

This works with Docker, Docker-Compose, Kubernetes and OpenShift out of the box.  Kubernetes and OpenShift would utilize FluentBit as either a daemon set (for global logging), or as a sidecar container (for application-level logging).

## Acknowledgments

Much sourced from [kelson-martins/docker-logging-fluentd
](https://github.com/kelson-martins/docker-logging-fluentd).
