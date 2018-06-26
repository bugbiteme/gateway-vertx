# gateway-vertx

This microservice can only be run locally if the inventory and catalog services are also running locally

* Inventory github project: [https://github.com/bugbiteme/inventory-wildfly-swarm](https://github.com/bugbiteme/inventory-wildfly-swarm)
* Catalog github project: [https://github.com/bugbiteme/catalog-spring-boot](https://github.com/bugbiteme/catalog-spring-boot)

The gateway microservice also cannot be run in OpenShift unless the inventory and catalog services are also running in OpenShift.

Instructions for deploying the entire coolstore application will be available soon.

## Run gateway-vertx as a stand-alone RestAPI service running locally

Since the API Gateway requires the Catalog and Inventory services to be running, start all three services simultaneously and verify that the API Gateway works as expected.

Open a new terminal window and start the Catalog service:

~~~~
$ cd catalog-spring-boot
$ mvn spring-boot:run
~~~~

Open another new terminal window and start the Inventory service:

~~~~
$ cd inventory-wildfly-swarm
$ mvn wildfly-swarm:run
~~~~

Now that Catalog and Inventory services are up and running, start the API Gateway service in a new terminal window:

~~~~
$ cd gateway-vertx
$ mvn vertx:run
~~~~

*You will see the following exception in the logs:* `java.io.FileNotFoundException: /.../kubernetes.io/serviceaccount/token`

*This is expected and is the result of Vert.x trying to import services form OpenShift. Since you are running the API Gateway on your local machine, the lookup fails and falls back to the local service lookup. It’s all good!*


validate it is running using curl (or a web browser)

`curl http://localhost:9001/8080/api/products`
 
 output

~~~~
[ {
  "itemId" : "329299",
  "name" : "Red Fedora",
  "desc" : "Official Red Hat Fedora",
  "price" : 34.99,
  "availability" : {
    "quantity" : 35
  }
},
...
]
~~~~

terminate service `ctrl-c` in all open terminals

## Build and deploy gateway service on OpenShift. 

Assuming you have logged into OpenShift, make sure you are in the coolstore project:

`$ oc project coolstore`

Launching the gateway requires an additional step, so that it can talk to the other services that it depends on. 

Vert.x service discovery integrates into OpenShift service discovery via OpenShift REST API and imports available services to make them available to the Vert.x application. Security in OpenShift comes first and therefore accessing the OpenShift REST API requires the user or the system (Vert.x in this case) to have sufficient permissions to do so. All containers in OpenShift run with a serviceaccount (by default, the project `default` service account) which can be used to grant permissions for operations, for instance, accessing the OpenShift REST API. 

Grant permission to the API Gateway to be able to access OpenShift REST API and discover services.

`$ oc policy add-role-to-user view -n coolstore -z default`

### Deploying from local source using Maven
To build and deploy the gateway service into OpenShift using the fabric8 maven plugin, run the following Maven command:

`$ mvn fabric8:deploy`

While you are waiting for the deploy command to complete, you can log into the OpenShift web consol and check the progress of your deployment, and even view the build and deployment logs, which should look very similar to the messages seen when running the service locally.

### Deploying from github
The Java S2I image enables developers to automatically build, deploy and run java applications on demand, in OpenShift Container Platform, by simply specifying the location of their application source code or compiled java binaries. In many cases, these java applications are bootable “fat jars” that include an embedded version of an application server and other frameworks (wildfly-swarm in this instance). 

Deploy the service from github

`$ oc new-app https://github.com/bugbiteme/gateway-vertx.git --name gateway --image-stream=redhat-openjdk18-openshift`

A build gets created and starts building the Node.js Web UI container image. You can see the build logs using OpenShift Web Console or OpenShift CLI:

`$ oc logs -f bc/gateway`

In order to access the gateway from outside (e.g. from a browser), it needs to get added to the load balancer. Run the following command to add the gateway service to the built-in HAProxy load balancer in OpenShift.

~~~
$ oc expose svc/gateway
$ oc get route gateway
~~~

While you are waiting for the deploy command to complete, you can log into the OpenShift web consol and check the progress of your deployment, and even view the build and deployment logs, which should look very similar to the messages seen when running the service locally.

### In the event of no jdk image
Some versions of minishift do not include the openjdk image stream. In this case, we need to use the Java S2I image to tell OpenShift how to find it. This is done by creating an image stream. The image stream definition can be downloaded here and used. To add the image stream to your project run the following command:

`$ oc create -f openjdk-s2i-imagestream.json`


### Validate 
Once the service has been deployed, you can get the url by running

`$ oc get route`

validate it is running using curl (or a web browser)

`curl http://SERVICE_ROUTE_OPENSHIFT_URL/api/products`
 
 output
 
~~~~
[ {
  "itemId" : "329299",
  "name" : "Red Fedora",
  "desc" : "Official Red Hat Fedora",
  "price" : 34.99,
  "availability" : {
    "quantity" : 35
  }
},
...
]
~~~~

Original source:
[http://guides-cdk-roadshow.b9ad.pro-us-east-1.openshiftapps.com/index.html#/workshop/roadshow/module/vertx]()


