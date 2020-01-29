# Caching Micronaut microservices on Kubernetes using Hazelcast

[Micronaut](https://micronaut.io/) is a modern, JVM-based, full stack microservices framework designed for building modular, easily testable microservice applications. It comes with different options to share data among the running services, such as caching.  

In this blog post, we will find out how we can implement caching through Micronaut microservices using Hazelcast with minimal configuration changes. We will also display how it runs on Kubernetes environment with a simple application. 

## Requirements

- Apache Maven to build and run the project.
- A containerization software for building containers. We will use [Docker](https://docs.docker.com/install/) in this guide. 
- A Kubernetes environment. We will use local `minikube` environment as k8s for demonstration. 

## Caching on Micronaut Sample

This guide contains a basic Micronaut microservice code sample using [Hazelcast IMDG](https://hazelcast.org/). You can see the whole project [here](https://github.com/hazelcast-guides/caching-micronaut-microservices-on-kubernetes/tree/master/final) and start building your app in the final directory. However, this guide will start from an initial point and build the application step by step. 

### Getting Started

Firstly, you can clone the Git repository below to your local. This repository contains two directories, `/initial` contains the starting project that you will build upon and `/final` contains the finished project you will build.

```
$ git clone https://github.com/hazelcast-guides/caching-micronaut-microservices-on-kubernetes.git
$ cd caching-micronaut-microservices-on-kubernetes/initial/
```

### Running the Micronaut Application 

The application in the initial directory is a basic Micronaut app having 3 endpoints: 

- `/` is the homepage returning “Homepage” string only
- `/put` is the mapping where key and value is saved to a local map through `@CachePut` annotation.
- `/get` is the mapping where the values in the local map can be obtained by keys through `@Cacheable` annotation. 

You can run this application using the commands below:

```
$ mvn clean package
$ java -jar target/caching-micronaut-microservices-on-kubernetes-0.1.0.jar
```

Now your app is running at `localhost:8080`. You can test it by using the following on another console: 

```
$ curl "localhost:8080"
$ curl "localhost:8080/put?key=myKey&value=hazelcast"
$ curl "localhost:8080/get?key=myKey"
```

The output should be similar to the following:

```
{"value":"hazelcast"}
```

The value returns as `hazelcast` since we put it in the second command, and `podName` returns `null` because we are not running the application in k8s environment yet. After the testing, you can kill the running application on your console.

### Running the App in a Container

To create the container (Docker) image of the application, we will use [Jib](https://github.com/GoogleContainerTools/jib) tool. It allows to build containers from Java applications without a Docker file or even changing the `pom.xml` file. To build the image, you can run the command below:

```
$ mvn clean compile com.google.cloud.tools:jib-maven-plugin:1.8.0:dockerBuild
```

This command will compile the application, create a Docker image, and register it to your local container registry. Now, we can run the container using the command below:

```
$ docker run -p 5000:8080 caching-micronaut-microservices-on-kubernetes:0.1.0
```

This command runs the application and binds the local `5000` port to the `8080` port of the container. Now, we should be able to access the application using the following commands:

```
$ curl "localhost:5000"
$ curl "localhost:5000/put?key=myKey&value=hazelcast"
$ curl "localhost:5000/get?key=myKey"
```

The results will be the same as before. You can kill the running application after testing. Now, we have a container image to deploy on k8s environment.

### Running the App in Kubernetes Environment

To run the app on k8s, we need a running environment. As stated before, we will be using `minikube` for demonstration. After this point, we presume that your k8s environment is running without any issues.

We will use a deployment configuration which builds a service with two pods. Each of these pods will run one container which is built with our application image. You can see our example configuration file (named as `kubernetes.yaml`) in the repository. Please click [here](https://github.com/hazelcast-guides/caching-micronaut-microservices-on-kubernetes/blob/master/final/kubernetes.yaml) to download this file. You can see that we are also setting an environment variable named `MY_POD_NAME` to reach the pod name in the application.

After downloading the configuration file, you can deploy the containers using the command below:

```
$ kubectl apply -f kubernetes.yaml
```
Now, we should have a running deployment. You can check if everything is alright by getting the pod list: 

```
$ kubectl get pods
```

It is time to test our running application. But first, we need the the IP address of the running k8s cluster to access it. Since we are using `minikube`, we can get the cluster IP using the command below:

```
$ minikube ip
```

We have the cluster IP address, thus we can test our application:

```
$ curl "http://[CLUSTER-IP]:31000/put?key=myKey&value=hazelcast"
$ while true; do curl [CLUSTER-IP]:31000/get?key=myKey;echo; sleep 2; done
```
The second command makes a request in a loop in order to see the responses from both pods. At some point, you should see a result similar below:

```
{"podName":"hazelcast-micronaut-statefulset-0"}
{"value":"hazelcast","podName":"hazelcast-micronaut-statefulset-1"}
```

This means the pod named `hazelcast-micronaut-statefulset-1` got the put request, and stored the value in its local cache. However, the other pod couldn't get this data and displayed `null` since there are no distributed caching between pods. To enable distributed caching, we will use Hazelcast IMDG in the next step. Now, you can delete the deployment using the following command:

```
$ kubectl delete -f kubernetes.yaml
```

### Caching using Hazelcast

To configure caching with Hazelcast, firstly we will add some dependencies to our `pom.xml` file:

```
<dependency>
    <groupId>io.micronaut.cache</groupId>
    <artifactId>micronaut-cache-hazelcast</artifactId>
    <version>${micronaut-cache-hazelcast.version}</version>
</dependency>
<dependency>
    <groupId>com.hazelcast</groupId>
    <artifactId>hazelcast</artifactId>
    <version>${hazelcast.version}</version>
</dependency>
<dependency>
    <groupId>com.hazelcast</groupId>
    <artifactId>hazelcast-kubernetes</artifactId>
    <version>${hazelcast-kubernetes.version}</version>
</dependency>
```

The first dependency is for Micronaut Cache for Hazelcast, the second one is for Hazelcast IMDG itself, and the last one is for Hazelcast's Kubernetes Discovery Plugin which helps Hazelcast members to discover each other on a k8s environment. Now, we just need to add a configuration bean to enable Hazelcast:

```
@Singleton
public class HazelcastAdditionalSettings
        implements BeanCreatedEventListener<HazelcastMemberConfiguration> {

    public HazelcastMemberConfiguration onCreated(BeanCreatedEvent<HazelcastMemberConfiguration> event) {
        HazelcastMemberConfiguration configuration = event.getBean();
        configuration.getGroupConfig().setName("micronaut-cluster");
        JoinConfig joinConfig = configuration.getNetworkConfig().getJoin();
        joinConfig.getMulticastConfig().setEnabled(false);
        joinConfig.getKubernetesConfig().setEnabled(true);
        return configuration;
    }
}
```  

This bean creates a `HazelcastAdditionalSettings` object to configure Hazelcast members. We enable the k8s config for discovery. 

Our application with Hazelcast caching is now ready to go. We do not need to change anything else because we are already using Micronaut caching annotations in `CommandService` class. 

### Running the App with Hazelcast IMDG in Kubernetes Environment 

Before deploying our updated application on k8s, you should create a `rbac.yaml` file which you can find it [here](https://github.com/hazelcast-guides/caching-micronaut-microservices-on-kubernetes/blob/master/final/rbac.yaml). This is the role-based access control (RBAC) configuration which is used to give access to the Kubernetes Master API from pods. Hazelcast requires read access to auto-discover other Hazelcast members and form Hazelcast cluster. After creating/downloading the file, apply it using the command below:

```
$ kubectl apply -f rbac.yaml 
```

Now, we can build our container image again and deploy it on k8s:

```
$ mvn clean compile com.google.cloud.tools:jib-maven-plugin:1.8.0:dockerBuild
$ kubectl apply -f kubernetes.yaml 
```

Our application is running, and it is time to test it again: 

```
$ curl "http://[CLUSTER-IP]:31000/put?key=myKey&value=hazelcast"
$ while true; do curl [CLUSTER-IP]:31000/get?key=myKey;echo; sleep 2; done
```
 
Since we configured distributed caching with Hazelcast, we can see the output similar to the following:

```
{"value":"hazelcast","podName":"hazelcast-micronaut-statefulset-0"}
{"value":"hazelcast","podName":"hazelcast-micronaut-statefulset-1"}
```

Now, both of the pods have the same value and our data is retrieved from Hazelcast distributed cache through all microservices!

## Conclusion

In this guide, we first developed a simple microservices application which uses caching with Micronaut annotations. The usage is very simple, but the cached data is not accessible from all microservices on a k8s environment if we don't use a distributed cache. In order to enable distributed caching, we used Hazelcast IMDG. The configuration is easy, just by adding a configuration bean. In the end, we succeeded to enable distributed caching among pods which helped us to access the same cached data from all microservices.