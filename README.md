# spring-smtp-gateway
The Spring SMTP Gateway is a set of applications that implement a lightweight SMTP server (`smtp-gateway`) which sends incoming RFC822 compliant messages over a Spring Cloud Stream destination to an arbitrary downstream processor application.  The processor application in this repo (`smtp-sink`) reads messages from the stream and dumps the full RFC822 message (including messages headers) to the standard out.


## Connectivity Configuration

The SMTP server is pre-configured to listen on TCP port 1026, but can be overridden using the following Spring configuration property:

```
smtpmqgateway.binding.port
```

The server is also pre-configured for a maximum messages size of 39845888 bytes and a maximum header size of 262144 bytes.  These can be overridden with the following properties:

```
smtpmqgateway.message.maxHeaderSize
smtpmqgateway.message.maxMessageSize

```

If you wish to limit the IPs that can connect to the server, a configuration option is available for an `allow list` of CIDR ranges that can connect to the server.  CIDR ranges are comma delimited.  The following is an example of a configured list of CIDR ranges:

```
smtpmqgateway.clientwhitelist.cidr=127.0.0.1/16,192.168.0.1/16
```

## Spring Cloud Streams Configuration

The applications services are pre-configured to use a RabbitMQ binding and by default attempt to connect to `localhost:5672` with a username\password of `guest\guest`.  These can be overriden using standard [Spring configuration properties](https://docs.spring.io/spring-boot/docs/current/reference/html/application-properties.html#appendix.application-properties.integration) for RabbitMQ.


