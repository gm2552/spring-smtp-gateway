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


## RabbitMQ

The default build configuration requires a RabbitMQ cluster to function properly.

### RabbitMQ Instalation

If you do not already have a RabbitMQ cluster running, you will spin one up.  Below will covers a few options.

#### Kubernetes Operator

The RabbitMQ Kubernetes operator provides a resource based option for deploying RabbitMQ clusters to a Kubernetes cluster.  It also has the nicety of supporting the Kubernetes [Service Binding Spec](https://github.com/servicebinding/spec).  To install the RabbitMQ operator, run the following command against your cluster. 

```
kubectl apply -f "https://github.com/rabbitmq/cluster-operator/releases/latest/download/cluster-operator.yml"
```

After the operator has been successfully installed, you can create a cluster by applying a RabbitMQCluster resource to you Kubernetes cluster.  The following is an example RabbitMQCluster resource that will spin up a cluster with 1 node in the default namespace.

```
apiVersion: rabbitmq.com/v1beta1
kind: RabbitmqCluster
metadata:
  name: rmq-1
spec:
  replicas: 1
```

If you are using a Tanzu Application Platform accelerator as described in a later section of this document, a RabbitMQCluster yaml file will be generated for you, and you simply need to run `kubectl apply -f	` using that generated file. 

### Connectivity Configuration

The application services do not care where or how the RabbitMQ cluster is installed; they just need configuration information to connect to the cluster.  As described earlier in this document, the configuration is done through Spring configuration properties, and there are many options to provided those properties.

#### Kubenetes Service Binding

If you are using Tanzu Application Platform or some other build system like `pack` or `kpack` to build a container, these system will automatically include the spring service binding library in the image.  The service binding library looks for mounted secrets in the container file system at runtime to obtain the necessary connectivity information.  Depending on your platform, those secrets are mounted using your platforms implementation of the service binding specification.

If you are using the Tanzu Application Platform, you can use the [services toolkit](https://docs.vmware.com/en/Services-Toolkit-for-VMware-Tanzu-Application-Platform/0.6/svc-tlk/GUID-overview.html) to create resource claims and attach those claims to the application services.  If you are using a Tanzu Application Platform accelerator as described in a later section of this document, the necessary yaml configuration will be generated for you, and you simply need to run `kubectl apply -f	` using that generated file.  A workload.yaml file will also be generated that will include the necessary `resource claim` configuration to bind to the RabbitMQ instance.

## Tanzu Application Platform Accelerator

This repository includes an `accelerator.yaml` file that is used by the Tanzu Application Platform [Accelerator](https://docs.vmware.com/en/Tanzu-Application-Platform/1.1/tap/GUID-tap-gui-plugins-application-accelerator.html) feature.  Using this repository as the backing for an accelerator, you can generate project and configuration files to facilitate connecting to necessary data services (eg. RabbitMQ) and building/deploying the application services.

