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

### Configuration Options

The accelerator contains the following configuration options:

* **SMTP Gateway Container Port:**  The port that the smtp-gateway micro-service will be listening on for SMTP connections and SHOULD match the application's port configuration.  The default port is 1026 which is the same default port that the application listens on.
* **SMTP Gateway Service Port:** The port that the Kubernetes service resource will be listening on.
* **RabbitMQ Cluster Name:**  The name of the RabbitMQ cluster resource that the micro-services will connect to.  If a new RabbitMQ cluster resource is to be created, this will be the name of the resource.  This name will also be propagated to the `workload.yaml` files in the resource claim section to indicate the names of the cluster that micro-services should connect to.
* **RabbitMQ Service Name:** The namespace where the RabbitMQ is deployed or where it will be deployed.  It is assumed that this namespace has already been created.
* **Workload Namespace:** The namespace where the application micro-services will be deployed.  It is assumed that this namespace has already been created.
* **Create RabbitMQ Cluster:** If this box is checked, the accelerator will generate a file named `rmqCluster.yaml` in the K8s directory that contains the resource definition for the cluster that will be created.
* **Number of RabbitMQ Nodes:** If a cluster needs to be created, this is the number of replica nodes that will be created.
* **Create Resource Claim:** If this box is checked, the accelerator will generate a file named `rmqResoruceClaim.yaml` in the K8s directory that contains the resource definition for the resource claims that the micro-services can use to create service bindings to the RabbitMQ cluster.  It also creates resource definitions used by the Tanzu CLI to manage service instances and resource claims. 

The generated zip file from the accelerator will contain project folders for all micro-services and yaml configuration files for the selected options.  It will also contain updated workload.yaml files that contain configuration data from the choices above.

### Application Deployment

Before deploying the application, you should first deploy any requested configuration items from the section above using the `kubectl` command against each yaml file.  

Ex:

```
kubectl apply -f service.yaml -f rmqCluster.yaml -f rmqResoruceClaim.yaml
```

Each project folder contains a `workload.yaml` file under its `config` folder.  If you are using the `tanzu cli`, you can use these files to build and deploy the micro-services.

Ex:

```
tanzu apps workloads create -f ./smtp-gateway/config/workload.yaml
```

Optionally, the accelerator creates a `workloads.yaml` in the K8s directory that can be used to deploy all micro-services.  This can be applied using the `kubectl` command.

```
kubectl apply -f workloads.yaml
```

## Testing the Deployment

Assuming the application has successfully deployed, you can test the application by using the `telnet` (or something else which can
speak SMTP) application and kubectl port forwarding.

In one command shell, run the following command substituting the local port with a port of you choosing and the container port and workload namespace from your selections in the accelerator.

```
kubectl port-forward deploy/smtp-gateway <local_port>:<container_port> -n <workload_namespace>
```

Once port forwarding is in place, you can use the the telnet command below in another command shell (substituting the local port with the local port in the previous step).

```bash
cat <<EOF | telnet localhost <local_port>
ehlo console
mail from: ea@vmware.com
rcpt to: gm@vmware.com
data
To: Evan Anderson <ea@vmware.com>
From: Greg Meyer <gm@vmware.com>
Subject: Application Test
MIME-Version: 1.0
message-id: 0c796d0e-4c76-43e8-be40-2cd5e30c1006
Date: Mon, 23 May 2023 07:57:27 -0500

Hello world!  I made it here.
.
quit
EOF
```

You can then check that the smtp-sink application received the message with the following command:

```
kubectl logs -l app.kubernetes.io/component=run,carto.run/workload-name=smtp-sink --tail 20 -n <workload_namespace>
```
