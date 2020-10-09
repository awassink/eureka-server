# Running Eureka Server and Clients on AWS ECS Fargate

I have created three projects to show how you can run Eureka server <https://github.com/awassink/eureka-server>, a Spring Boot hello world service <https://github.com/awassink/eureka-hello-service> registering itself in Eureka and a Spring Boot backend for frontend proxy service <https://github.com/awassink/eureka-b4f-service> looking up the hello world service in Eureka.
I based this solution on the Eureka Fargate Sample project of Jussi Seppälä <https://github.com/jussiseppala/eurekafargatesample>.
 
In my case I use an ALB to have a stable endpoint to connect to the Eureka server.
The deployment of the eureka server is in the way a pretty straight forward Service deployment in ECS Fargate.

The main problem with Eureka in ECS Fargate lies in the registering of the Eureka clients (the services) in the Eureka server.
The services need to be registered with the internal IP-address assigned to the Fargate Task.
By default the IP-address of the Docker container within the Fargate task is used for the registration.
This container IP-address is of course useless because there is no network route from outside of the Fargate Task to this IP-address.
The private IP-address of a Fargate Task is dynamically assigned on creation of the Task, so we can't configure it via a property, but we need to determine it dynamically on startup of the Spring Boot application.
This is where the `CloudConfiguration` class created by Jussi steps in.
It determines the Task private IP-address using the ECS Task Metadata service to get the correct IP and use that when registering to Eureka.

## Eureka Server deployment

Prerequisite is that you have an ECS-cluster with an ALB available for the deployments. 

Create the Docker image of the `eureka-server` project using the maven build followed by the Docker build.
Create an ECR repository for the `eureka-server` and push the created image to the ECR repository.
Create the ECS Task Definition named `eureka-server` for Fargate and set a IAM Role to use for Tasks for in this definition.
Make shure that the selected IAM Role has permissions to use ECR.
Set the Task size to `0.5GB` and `0.25 vCPU`, which should be enough.
Add the `eureka-server` container and set the `image` to the image created in the ECR repository and set the `Port mappings` to `8761`.
Now create the `eureka-server` ECS Fargate Service based on the created Template.
Select the ALB Load Balancer and 'add a loadbalancer' with a new listener for port `8762`.
Set the `Health check path` to `/actuator/health`.
Change the SecutityGroups of the ALB and the `eureka-server` Task to allow inbound traffic on port `8761`.

The `eureka-server` should now become available on your ALB: `http://<alb-hostname>:8761/`

## Hello- and B4F Service deployment

