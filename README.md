# Spring Cloud Config : Using Git Webhook to Auto Refresh the config changes with Spring Cloud Stream, Spring Cloud Bus and RabbitMQ

![The architecture](https://chathurangat.files.wordpress.com/2018/07/untitled-diagram-10.png "The architecture")

Whenever the update is pushed to the Git Repository,  it will send the Webhook event to the registered application. That is the /monitor endpoint of the Spring Cloud Config Server.

Then the Spring Cloud Config Server will retrieve the latest configuration property changes from the Git repository and publish the refresh event to the Spring Cloud Bus. This refresh event is published with Spring Cloud Stream.

All the distributed application services will connect to Spring Cloud Bus and will listen for the refresh event published by the Spring Cloud Config Server.  The Spring Cloud Bus will broadcast the refresh event across all connected application services. Therefore it is guaranteed that the the published refresh event will be received by every distributed service (Config Client) that is connected to the Spring Cloud Bus.

Every Config Client (application service) has the Spring Boot Actuator in its classpath. Therefore all the application services can accept and handle the refresh event with no issue. Then all the beans annotated with @RefreshScope will be refreshed and the properties will be re-fetched. That means the Config Client will communicate with Config Server to retrieve the latest configuration properties for the related to the annotated beans.

The Config Server pulls the latest configurations from the Git repository (property source) and updates the Config Sever itself.  After the Config Client requests for getting the properties will be served with latest updated properties.

Finally the Config Client will receive the latest and updated properties through the Config Server.

## Installing RabbitMQ with Docker

Let’s start with RabbitMQ, which we recommend running as RabbitMQ as a docker image. First we need to install Docker and run following commands once Docker is installed successfully:

`docker pull rabbitmq:3-management`

This command pulls RabbitMQ docker image together with management plugin installed and enabled by default.

Next, we can run RabbitMQ:

`docker run -d --hostname my-rabbit --name some-rabbit -p 15672:15672 -p 5672:5672 rabbitmq:3-management`

Once we executed the above command, we can go to the web browser and open http://localhost:15672, which will show the management console login form.

The default username is: ‘guest’; password: ‘guest’.

RabbitMQ will also listen on port 5672. (but the web port is 15672)

## spring-cloud-config-monitor

spring-cloud-config-monitor provides a /monitor endpoint for the Config Server to receive notification events when the properties backed by a Git repository are changed. This will work if and only if spring.cloud.bus property is enabled.

Many source code repository providers (such as Github, Gitlab, or Bitbucket) notify you of changes in a repository through a webhook. You can configure the webhook through the provider’s user interface as a URL and a set of events in which you are interested. For instance, Github uses a POST to the webhook with a JSON body containing a list of commits.

If you add a dependency on the spring-cloud-config-monitor library and activate the spring.cloud.bus.enabled property in your Config Server, then the /monitor endpoint will be enabled.  This can be done by adding the following entry in the application.properties of the Spring Cloud Config Server.

`spring.cloud.bus.enabled = true  #add to the application.properties of Config Server`

By default spring.cloud.bus.enabled is set to false, meaning the Spring Cloud Config Server won’t use Spring Cloud Bus capabilities to process Git push events notifications.

When the webhook is activated, the Config Server sends a refresh event targeting the applications that the property changes should be reflected. (e.g:- RefreshRemoteApplicationEvent)

By default, it looks for changes in files that match the application name (for example, department.properties is targeted at the department application, while application.properties is targeted at all applications).

## Spring Cloud Stream  (spring-cloud-starter-stream-rabbit)

Spring Cloud Stream is a framework that supports in developing message driven or event driven microservices.  spring-cloud-starter-stream-rabbit is a specific implementation of Spring Cloud Stream that uses RabbitMQ message broker as underlying message broker.

e.g:- If you want to use the Kafka as the underlying message broker, then you have to use the dependency spring-cloud-starter-stream-kafka instead of this.

spring-cloud-starter-stream-rabbit is used to send/publish event notifications from the Config Server to a RabbitMQ exchange (again, only if spring.cloud.bus property is enabled). The Spring Cloud Bus will broadcast the event (refresh event) to all related services.

Since the Spring Cloud Stream is published the event to the RabbitMQ, it should have the related connection details in application.properties. Therefore add the following RabbitMQ connection details to the application.properties of the Spring Cloud Config Server. (Change the connection details according to your RabbitMQ message broker)

    spring.rabbitmq.host=localhost
    spring.rabbitmq.port=5672
    spring.rabbitmq.username=guest
    spring.rabbitmq.password=guest

## Adding the Webhook event with GitHub

You can add the  /monitor endpoint URL as the webhook URL of your repository.  No that, this will not work for localhost domain. You should have a public domain name or public ip address.

## Auto refreshing with webhook event

Once the Git repository received the pushed update, it will notify the given URL endpoint with set of parameters (The URL will be registered during the webhook registration process). This is known as triggering the webhook event.

If you do not have a public domain or ip address mapped to your /monitor endpoint, the GitHub webhook event will not work.  In this case, you have to simulate the webhook event as follows.