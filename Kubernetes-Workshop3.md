[Previous Page](Kubernetes-Workshop2.md) / [Next Page](Kubernetes-Workshop4.md)
---
***Note: This contents is Beta version!!***

---
* [Azure 無料アカウントのご取得はコチラから](https://aka.ms/jjug_mar2)  
***Azure の無料アカウントで料金は発生しますか？  
いいえ。無料で開始でき、最初の 30 日間に使用する ¥22,500 クレジットを取得します。サービスの利用を開始した後でも、アップグレードするまでは無料で利用できます。***  

---

# 3. Basic operation to create and publish the Service

I uploaded a tiny sample Java project on the GitHub as 
 [AccountSvc sample](https://github.com/yoshioterada/k8s-Azure-Container-Service-AKS--on-Azure/tree/master/AccountSvc). Please refer to the detail configuration on the sample project

## 3.01. Create Azure Container Registry (Private Container Registry)

At first, you need to create a Private Container Registry. If you didn't create it, at first please create it?   

I wrote the explanation how to create the [Create Azure Container Registry,  please refer to it?](CreateAzureContainerRegistry.md)

## 3.0.2 Prepare Source Code & Dockerfile (Multi-Stage Build)

At first, Please create Maven project like follows.

```
$ pwd
/Users/HOME_DIRECTORY/Projects/AccountSvc
$ ls
pom.xml     src
```

After created the Maven project, please create the Dockerfile on the same directory as the above Maven project.

```
#####################################################
# Maven Build & Test & Create an artifact
#####################################################
FROM maven:3.5-jdk-8 as BUILD
MAINTAINER Yoshio Terada

COPY hazelcast-default.xml /usr/src/myapp/hazelcast-default.xml
COPY src /usr/src/myapp/src
COPY pom.xml /usr/src/myapp
RUN mvn -f /usr/src/myapp/pom.xml clean package

#####################################################
# Build container image copying from BUILD artifact
#####################################################
# FROM java:8-jdk-alpine
FROM openjdk:8-jre-alpine 
MAINTAINER Yoshio Terada

COPY --from=BUILD /usr/src/myapp/hazelcast-default.xml /tmp
COPY --from=BUILD /usr/src/myapp/target/payara-micro.jar /tmp
COPY --from=BUILD /usr/src/myapp/target/AccountSvc-1.0-SNAPSHOT.war /tmp/app.war

RUN addgroup -g 1000 java
RUN adduser -D -u 1000 -G java java
RUN chown java:java /tmp/payara-micro.jar
RUN chmod 755 /tmp/payara-micro.jar

USER java

EXPOSE 8080
ENTRYPOINT java -jar /tmp/payara-micro.jar --hzconfigfile /tmp/hazelcast-default.xml --deploy /tmp/app.war
```
Note: There is two ***FROM*** line on the above Docker file. This is based on the [Multi-Stage Build (Docker 17.05 or higher)](https://docs.docker.com/develop/develop-images/multistage-build/#use-multi-stage-builds). If you create the above, you can apply the Builder Pattern.  

Please create thd Dockerfile on the same directory with Maven project directory. If you create the Dockerfile, you are able to build the source code and test, after that you can create the docker images.

This is an example of PayaraMicro, however you will be able to create similar file on such like WildFly, TomEE, Liberty and SpringBoot.

***For Java SE 9 :***  
In the above example, I used Java SE 8. However we can create custom JRE since Java SE 9. In order to create more small size of Docker image, please consider to use the Java SE 9 or latest?

* [Slim modular Java 9 runtime Docker image with Alpine Linux](https://blog.jdriven.com/2017/11/modular-java-9-runtime-docker-alpine/)  
* [Building tiny docker containers with JDK9](https://blog.dekstroza.io/building-minimal-docker-containers-with-java-9/)  



## 3.1 Create Deployment manifest(YAML) file

The minimum configuration for Deployment will be look like follows.

```
apiVersion: apps/v1beta2
kind: Deployment
metadata:
  name: account-service
  namespace: order-system-production
spec:
  replicas: 2
  template:
    metadata:
      labels:
        app: account-service
        env: test
        version: v1
    spec:
      containers:
      - name: account-service
        image: yoshio.azurecr.io/tyoshio2002/account-service:1.14
```

However if you would like to provide the service on production environment, more additional configuration will be needed like follows.

```
apiVersion: apps/v1beta2
kind: Deployment
metadata:
  name: account-service
  namespace: order-system-production
spec:
  replicas: 2
  selector:
    matchLabels:
      app: account-service
  minReadySeconds: 60
  progressDeadlineSeconds: 600
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 50%
      maxUnavailable: 50%
  template:
    metadata:
      labels:
        app: account-service
        version: v1
    spec:
      imagePullSecrets:
        - name: docker-reg-credential
      securityContext:
        readOnlyRootFilesystem: true
      containers:
      - name: account-service
        image: yoshio.azurecr.io/tyoshio2002/account-service:1.28
        terminationGracePeriodSeconds: 60
        livenessProbe:
          httpGet:
            path: /healthz
            port: 8080
          initialDelaySeconds: 120
          timeoutSeconds: 5
          periodSeconds: 10
          failureThreshold: 3
        readinessProbe:
          httpGet:
            path: /healthz
            port: 8080
          initialDelaySeconds: 120
          timeoutSeconds: 5
          periodSeconds: 10
          failureThreshold: 3
        resources:
          limits:
            memory: "1Gi"
          requests:
            memory: "600Mi"
```
***Note: This manifest(yaml) file was created for v1.8.***


### 3.1.1 Rolling Upgrade

```
spec:
  minReadySeconds: 60
  progressDeadlineSeconds: 600
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 50%
      maxUnavailable: 50%
```

* ***minReadySeconds:***  
The bootup time of your application, Kubernetes waits specific time until the next pod creation.
For example minReadySeconds is 60 then after the Pod become healthy the Deployment must wait for 60 seconds to update the next Pod.

* ***progressDeadlineSeconds***  
Timeout value for updating one pod in this example 10 minutes. If the rollout fails to progress in 10 minutes, then the Deployment is marked as failed and finished, also all of next job never invoke.

* ***maxSurge:***  
The maxSurge parameter controls how many extra resources can be created during the rollout.(Absolute value or %)

* ***maxUnavailable:***  
The maxUnavailable parameter sets the maximum number of Pods that can be unavailable during a rolling update.(Absolute value or %)

### 3.1.2 Label is very important

Label is used for grouping, filtering, viewing, and operating.  
Labels has key/value pairs. The key name must be shorter than 63 characters.  Values is strings with a maximum length of 63 characters.

```
  template:
    metadata:
      labels:
        app: account-service
        version: v1
        stage: development
        experimental: true
```

### 3.1.3 Pull Image from Private Container Registry

In order to access and get the container image from private container registry (Azure Container Registry), you need to create the credential information. For example, if you would like to push the container image to the container private registry, you will execute following command to login.

```
$ docker login -u foo-bar ******.azurecr.io
Password: 
Login Succeeded
```

Same as before, in order to login to the private container registry, we need to create the credential infomation like follows before creating the Deployment. In order to create the credential, please execute following command and create the Secret?

```
$ kubectl create secret docker-registry docker-reg-credential \
  --docker-server=******.azurecr.io --docker-username=foo-bar \
  --docker-password=+********************/**********  \
  --docker-email=foo-bar@microsoft.com
secret "docker-reg-credential" created
```

After executed the above, you can confirm the credential information like follows.

```
$ kubectl get secret docker-reg-credential -o yaml
apiVersion: v1
data:
  .dockerconfigjson: *****************************************************
                    ******************************************************
                    ******************************************************
                    ******************************************************
                    ******************************************************
                    ******+ 
kind: Secret
metadata:
  creationTimestamp: 2018-03-07T05:23:28Z
  name: docker-reg-credential
  namespace: order-system-production
  resourceVersion: "13546"
  selfLink: /api/v1/namespaces/order-system-production/secrets/docker-reg-credential
  uid: aab0c43f-****-****-****-0a58ac1f1266
type: kubernetes.io/dockerconfigjson
```

Then you can refer to the credental incormation by name of "docker-reg-credential" in Secret like follows. And it is possilbe to get the image from private container registry.

```
    spec:
      imagePullSecrets:
        - name: docker-reg-credential
```

### 3.1.4 Liveness & ReadinessProbe Probe

```
        livenessProbe:
          httpGet:
            path: /healthz
            port: 8080
          initialDelaySeconds: 60
          timeoutSeconds: 1
          periodSeconds: 10
          failureThreshold: 3
```

The above manifest configured ***livenessProbes*** which confirm whether an container is running properly or not.

It probe the health check by using HTTP GET request to /healthz with port 8080.

The probe sets an ***initialDelaySeconds=60*** which means that it will not be called until 60 seconds after all the containers in the Pod are created.
And ***timeoutSeconds=1*** was configured it means that the probe must respond with in the 1 second timeout. 
The ***periodSeconds=10*** was configured, it means that the probe invoke every 10 seconds. If more than 3 probe failed(***failureThreshold=3***), the container will be considerd un-healthy and failed and restart.

```
        readinessProbe:
          httpGet:
            path: /healthz
            port: 8080
          initialDelaySeconds: 120
          timeoutSeconds: 5
          periodSeconds: 10
          failureThreshold: 3
```

The above ***readinessProbe*** is more important than liveness probe in production environment. the readinessProbe is confirm whether the service can acceptable or not. If this probe failed, the internal loadbalancer never send the traffic to this pod. Only successed the probe, the traffic to this pod will start.


### 3.1.5 Request Limitation

Always the application consume CPU and memory resouces. And it is important for us to specify the minimum and maximux resouces usages. If you execute the following command, you can confirm the current usage for both resources.

For example, you execute the high load test to the pod, you may be able to estimate the mimimum and maximux usage of the resources.

```
$ kubectl top pod account-service-77c48cc54-d9g58 
NAME                              CPU(cores)   MEMORY(bytes)   
account-service-77c48cc54-d9g58   72m          377Mi   
```

In order to specify the minimum usage(Default usage), you can specify the ***requests***.

```
          requests:
            cpu: "250m"
            memory: "500Mi" 
```

In order to specify the maximum usage, you can specify the ***limits***.

```
          limits:
            cpu: "500m"
            memory: "1Gi"
```

For CPU entry:  
The above 0.5(500m) is guaranteed to use the half CPU in 1 CPU.
The expression 0.1 is equivalent to the expression 100m, which can be read as “one hundred millicpu”. CPU is always requested as an absolute quantity, never as a relative quantity.

For Memory entry:   
You can express memory as a plain integer or as a fixed-point integer using one of these suffixes: E, P, T, G, M, K. You can also use the power-of-two equivalents: Ei, Pi, Ti, Gi, Mi, Ki. For example, the following represent roughly the same value:

[Managing Compute Resources for Containers](https://kubernetes.io/docs/concepts/configuration/manage-compute-resources-container/#meaning-of-cpu)

### 3.1.6 Init Container

If you would like to invoke or wait somethings before starting the POD image, you can create ***Init Container***.  
For example, before staring the pod, you need to check whether DB is running or not.

```
    spec:
      containers:
      - name: account-service
        image: yoshio.azurecr.io/tyoshio2002/account-service:1.14
      ....
      initContainers:
        - name: init-myservice
          image: busybox
          command: ['sh', '-c', 'for i in {1..100}; do sleep 1; if dig myservice; then exit 0; fi; done; exit 1']
        - name: init-mydb
          image: busybox
          command: ['sh', '-c', 'until nslookup mydb; do echo waiting for mydb; sleep 2; done;']
```

### 3.1.7 PostStart Hook of POD

The postStart hook is executed immediately after the container’s main process is started. After started the container,if you would like to execute some command, PostStart hook is available.

```
    spec:
      containers:
      - name: account-service
        image: yoshio.azurecr.io/tyoshio2002/account-service:1.14
        lifecycle:
          postStart:
            exec:
              command:
                - sh
                - -c
                - "echo 'hook will fail with exit code 15'; sleep 5; exit 15"    2
```

### 3.1.8 PreStop Hook of POD

The preStop hook is executed immediately before a container is terminated. When a container needs to be terminated, the k8s will run the preStop hook(if configured) and only then send a SIGTERM to the process.  
Before stop the container,if you would like to invoke some call, PostSttop hook is available.

```
    spec:
      containers:
      - name: account-service
        image: yoshio.azurecr.io/tyoshio2002/account-service:1.14
        lifecycle:
          preStop:
            httpGet:
              port: 8080
              path: shutdown
```

The above invoke HTTP GET request to http://POD_IP:8080/shutdown as soon as started the  terminating container.

```
    lifecycle:
      preStop:
        exec:
          command:
          - sh
          - -c
          - "sleep 60"
```

The above wait 60sec as soon as started the  terminating container.



## 3.2 Create Service manifest(YAML) file

After created the deployment, please create Service manifest to expose the Services. In the above Deployment, you specified the label. 


```
apiVersion: v1
kind: Service
metadata:
  labels:
    app: account-service
  name: account-service
  namespace: order-system-production
spec:
  ports:
  - port: 80
    name: http
    targetPort: 8080
  - port: 443
    name: https
    targetPort: 8080
  selector:
    app: account-service
  sessionAffinity: None
  type: ClusterIP
```


## 3.3 Create All-in-one manifest(YAML) file

Please create a file as ***create-deployment-svc.yaml*** and write the following manifest into the file?

```
apiVersion: apps/v1beta2
kind: Deployment
metadata:
  name: account-service
  namespace: order-system-production
spec:
  replicas: 2
  selector:
    matchLabels:
      app: account-service
  minReadySeconds: 60
  progressDeadlineSeconds: 600
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 50%
      maxUnavailable: 50%
  template:
    metadata:
      labels:
        app: account-service
        version: v1
    spec:
      imagePullSecrets:
        - name: docker-reg-credential
      securityContext:
        readOnlyRootFilesystem: true
      containers:
      - name: account-service
        image: yoshio.azurecr.io/tyoshio2002/account-service:1.28
        terminationGracePeriodSeconds: 60
        livenessProbe:
          httpGet:
            path: /healthz
            port: 8080
          initialDelaySeconds: 120
          timeoutSeconds: 5
          periodSeconds: 10
          failureThreshold: 3
        readinessProbe:
          httpGet:
            path: /healthz
            port: 8080
          initialDelaySeconds: 120
          timeoutSeconds: 5
          periodSeconds: 10
          failureThreshold: 3
        resources:
          limits:
            memory: "1Gi"
          requests:
            memory: "600Mi"
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: account-service
  name: account-service
  namespace: order-system-production
spec:
  ports:
  - port: 80
    name: http
    targetPort: 8080
  - port: 443
    name: https
    targetPort: 8080
  selector:
    app: account-service
  sessionAffinity: None
  type: ClusterIP
```

***Note: This manifest(yaml) file is created for v1.8.***

## 3.4 Build and Publish the services 


If you modify the source code, you need to execute mvn, docker build, docker tag, docker push, kubectl apply command every times to update the image.
To avoide multiple execute command every times, I created shell script like folllows.


```
#!/bin/bash
set -e 

if [ "$1" = "" ]
then
    echo "./build-create.sh [version-number]"
    exit 1
fi

export VERSION=$1

# Specify the docker image name
DOCKER_IMAGE=tyoshio2002/account-service
# Specify the Private Docker Registry name
DOCKER_REPOSITORY=yoshio.azurecr.io

# Build docker image
docker build -t $DOCKER_IMAGE:$VERSION .
docker tag $DOCKER_IMAGE:$VERSION $DOCKER_REPOSITORY/$DOCKER_IMAGE:$VERSION

# Push the image to Private Docker Registry
docker push $DOCKER_REPOSITORY/$DOCKER_IMAGE:$VERSION

# Change the version for docker image inside of manifest(YAML) file 
sed -i -e "s|image: .*|image: $DOCKER_REPOSITORY/$DOCKER_IMAGE:${VERSION}|g" create-deployment-svc.yaml

# Apply the new Image to the Service
/usr/local/bin/kubectl apply --record -f create-deployment-svc.yaml

# Delete the <none> image which is created during the build of maven
#
# REPOSITORY    TAG       IMAGE ID        CREATED                  SIZE
# <none>        <none>    883046e0d36f    Less than a second ago   1.3GB
#
docker images | awk '/<none/{print $3}' | xargs docker rmi 
```

After created the above shell script, you can execute like follows.

```
$ ./build-create.sh 
./build-create.sh [version-number]
```

***Note 1 :***  
You need to specify the ***DOCKER_REPOSITORY***, which you created at the [Create Azure Container Registry (Private Docker Registry)](https://github.com/yoshioterada/k8s-Azure-Container-Service-AKS--on-Azure/blob/master/CreateAzureContainerRegistry.md).

***Note 2:***  
Before executed the above build script, please confirm the files and directory structure again? It may look like follows. If you face some problems, please refer to my sample project as 
 [AccountSvc sample](https://github.com/yoshioterada/k8s-Azure-Container-Service-AKS--on-Azure/tree/master/AccountSvc).?


```
$ ls -F
Dockerfile			k8s-config-yamls/
build-create.sh*		nb-configuration.xml
create-deployment-svc.yaml	pom.xml
hazelcast-default.xml		src/
```


Finally please execute the build script command as follows?

```
$ ./build-create.sh 1.33
```

The build message lool like follows.

```
$ ./build-create.sh 1.33
Sending build context to Docker daemon  374.5MB
Step 1/19 : FROM maven:3.5-jdk-8 as BUILD
 ---> 83d235b52940
Step 2/19 : MAINTAINER Yoshio Terada
 ---> Using cache
 ---> 85fa502d479d
Step 3/19 : COPY hazelcast-default.xml /usr/src/myapp/hazelcast-default.xml
 ---> Using cache
 ---> ba5b696b4b78
Step 4/19 : COPY src /usr/src/myapp/src
 ---> 62f8ccb8e6a9
Step 5/19 : COPY pom.xml /usr/src/myapp
 ---> ec2d391e7be8
Step 6/19 : RUN mvn -f /usr/src/myapp/pom.xml clean package

....
....
....


Step 18/19 : EXPOSE 8080
 ---> Running in 155397935a56
Removing intermediate container 155397935a56
 ---> ec52b8bd3286
Step 19/19 : ENTRYPOINT java -jar /tmp/payara-micro.jar --hzconfigfile /tmp/hazelcast-default.xml --deploy /tmp/app.war
 ---> Running in d438cdf4cc33
Removing intermediate container d438cdf4cc33
 ---> f9aebdd3d83a
Successfully built f9aebdd3d83a
Successfully tagged tyoshio2002/account-service:1.33
The push refers to repository [yoshio.azurecr.io/tyoshio2002/account-service]
8ba3ac19bd4b: Pushed 
9a11ee755403: Pushed 
107495bd2c60: Pushed 
12b3735e5ec4: Pushed 
d5aa3c00f012: Pushed 
29a52d69e152: Layer already exists 
a6dbce61bbcb: Layer already exists 
638d4576a926: Layer already exists 
c9b26f41504c: Layer already exists 
cd7100a72410: Layer already exists 
1.33: digest: sha256:2066b6a4eea5ed463549cec58c71d3a8581466b46d0cb216699014e89ba4d915 size: 2417
deployment "account-service" configured
service "account-service" unchanged
Deleted: sha256:883046e0d36f863c44b90502188aa2867eef47894fc7b8226a0d5bbce4ef64df
Deleted: sha256:38986db6f23b4e72cbd4d94c768d848f2b0c9dd1e1e02ec5a735e37608255f8d
Deleted: sha256:0f25cea1cd6046b27b3b279bad2c221b4fbee3b87c5ee138080e0193f384dfe9
Deleted: sha256:bd82694cfacbdd42611ad2cfc6ccd7f6ebf4a5f09b72676f249f5c5e64916d57
Deleted: sha256:bc1ce8c4f1c7e5c37ed310e37a6eda3a228958bb3170c0b8b3e116e3a59a486c
Deleted: sha256:5231fa027dd9bf94819f0e1649076a135041e4c13c576dbd8848deff7b220836
Deleted: sha256:cec3cdee9175a59f029e970e3eaa634bb5bf4ad69ebd0fd858d2931a4cf8dc9c
Deleted: sha256:bdbb6b75ce8cd61efa907e5b594450c35bfc0485175a0d990f71e4210d5ef135
Deleted: sha256:35546bfb36ff32991e78482d063fa471b41bdf256ddf370f2e36893b6083f256
Deleted: sha256:9bbccaa33b6cf25b701b17c77517b23d42c7b3eec3f64fc331e6dc430fff11e3
Deleted: sha256:6b819128e5739d837af9dea12272e1e1b4e05ea70d395aeb98ef457ebdc1634e
```

## 3.5 Confirm the deployment and services

```
$ kubectl get deploy,svc account-service  
NAME                     DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
deploy/account-service   2         2         2            2           20d

NAME                  TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)          AGE
svc/account-service   ClusterIP   10.0.206.8   <none>        80/TCP,443/TCP   20d
```

```
$ kubectl get po |grep account
NAME                                          READY     STATUS    RESTARTS   AGE
account-service-2573264708-1cz6p              1/1       Running   0          19m
account-service-2573264708-wffpv              1/1       Running   0          19m

```


## 3.6 How to access the Application (port-forward)

```
$ kubectl port-forward account-service-2573264708-1cz6p 8080:8080
Forwarding from 127.0.0.1:8080 -> 8080
```

Then you can access to the application by port-forwarding.

```
$ curl http://localhost:8080/app/account/rest/hello
$ curl http://localhost:8080/app/account/rest/listUsers
```

## 3.7 How to write YAML file?

If you don't understand how to write the YAML file, following command may help you. ***kubectl explain*** command will show the manual to write the entry like follows.

```
$ kubectl explain deployment.spec
RESOURCE: spec <Object>

DESCRIPTION:
     Specification of the desired behavior of the Deployment.

     DeploymentSpec is the specification of the desired behavior of the
     Deployment.

FIELDS:
   minReadySeconds	<integer>
     Minimum number of seconds for which a newly created pod should be ready
     without any of its container crashing, for it to be considered available.
     Defaults to 0 (pod will be considered available as soon as it is ready)

   paused	<boolean>
     Indicates that the deployment is paused and will not be processed by the
     deployment controller.

   progressDeadlineSeconds	<integer>
     The maximum time in seconds for a deployment to make progress before it is
     considered to be failed. The deployment controller will continue to process
     failed deployments and a condition with a ProgressDeadlineExceeded reason
     will be surfaced in the deployment status. Note that progress will not be
     estimated during the time a deployment is paused. This is not set by
     default.

   replicas	<integer>
     Number of desired pods. This is a pointer to distinguish between explicit
     zero and not specified. Defaults to 1.

   revisionHistoryLimit	<integer>
     The number of old ReplicaSets to retain to allow rollback. This is a
     pointer to distinguish between explicit zero and not specified.

   rollbackTo	<Object>
     DEPRECATED. The config this deployment is rolling back to. Will be cleared
     after rollback is done.

   selector	<Object>
     Label selector for pods. Existing ReplicaSets whose pods are selected by
     this will be the ones affected by this deployment.

   strategy	<Object>
     The deployment strategy to use to replace existing pods with new ones.

   template	<Object> -required-
     Template describes the pods that will be created. 
```

You can get the manual of child value too.

```
$ kubectl explain deployment.spec.minReadySeconds
FIELD: minReadySeconds <integer>

DESCRIPTION:
     Minimum number of seconds for which a newly created pod should be ready
     without any of its container crashing, for it to be considered available.
     Defaults to 0 (pod will be considered available as soon as it is ready)
```

---
[Previous Page](Kubernetes-Workshop2.md) / [Next Page](Kubernetes-Workshop4.md)
