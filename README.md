**Auther: Zelin (Zane) Wang**

**The Image URLs**  
Web-App:  
[https://hub.docker.com/r/zelinwang10/sentiment-analysis-web-app](https://hub.docker.com/layers/zelinwang10/sentiment-analysis-web-app/latest-amd64/images/sha256-848eea985d7b559456e8f0f220853ee4cfd00fd037dd70330337f026adfc9934?context=explore)  
Logic:  
[https://hub.docker.com/r/zelinwang10/sentiment-analysis-logic](https://hub.docker.com/layers/zelinwang10/sentiment-analysis-logic/latest-amd64/images/sha256-ab8b688c73e7ac0245da7181ba72a873a942cfad312662ea82598671778757e4?context=explore)  
Frontend:  
[https://hub.docker.com/r/zelinwang10/sentiment-analysis-frontend ](https://hub.docker.com/layers/zelinwang10/sentiment-analysis-frontend/latest-amd64/images/sha256-043ce429f3cae1936f29145855c5845c122f8e439ff648e52e8880c660aa6383?context=explore)  

**My Cloud Service Endpoint**  
Frontend: http://34.138.150.173:80/ (to access the application on the cloud)  
Logic: http://35.190.130.116:80/  
Web-App: http://35.196.48.152:80/  

**Demo Video on Google Drive**  
Demo Video: https://drive.google.com/file/d/1aqglTfHW5T4C_8Z7W5CCSGlhaLkkvtMj/view?usp=sharing  

**My Source Code**  
Check out my forked Repo (https://github.com/zelinewang/k8s-mastery) to see my modification.  

### Table of Contents

- [Git Repo](#git-repo)
  - Introduction
  - Application Functionality
  - Technical Perspective
  - Repo Structure
  - Further Instructions
- [Environment](#environment)
  - System Information
  - Required Software
- [Create Images](#create-images)
  - Building Process
  - Docker Commands
  - Platform Considerations
- [Immigrate Images to Cloud](#immigrate-images-to-cloud)
  - Docker Pull and Push
  - Google Cloud Platform Setup
  - Kubernetes Engine Configuration
- [Make the Deployment](#make-the-deployment)
  - Kubernetes Cluster Overview
  - YAML Configuration
  - Deployment Process
- [Port Numbers](#port-numbers)
  - Port vs Target Port
  - Analogy to understand
  - Accessing Services
- [Changes for Cloud Deployments](#changes-for-cloud-deployments)
  - Problem Identification
  - Source Code Modification
  - Rebuilding Images
- [Rebuild the Images and Modify the Deployment YAML File](#rebuild-the-images-and-modify-the-deployment-yaml-file)
  - Image Update
  - YAML Modification
  - Deployment Update
- [Conclusion](#conclusion)
- [README of the Original Repo](#readme-of-the-original-repo)


---

## Git Repo

First, we need to investigate the code in this repo that we gonna use: https://github.com/rinormaloku/k8s-mastery  
For more specific introduction and guidelines, we can go to this page ([Learn Kubernetes in Under 3 Hours: A Detailed Guide to Orchestrating Containers](https://www.freecodecamp.org/news/learn-kubernetes-in-under-3-hours-a-detailed-guide-to-orchestrating-containers-114ff420e882)) for instructions on building Docker images and local simple Kubernetes cluster ([minikube](https://minikube.sigs.k8s.io/docs/start/)) upon this Microservice-based application.  

But what we gonna do is to use Kubernetes ([GKE](https://console.cloud.google.com/kubernetes/)) to build this Microservice-based application on the Google Cloud Platform, and make the frontend services available to users and easy and carefree to update, scale up or down, and balance the workload, on ***Cloud***.  

Generally, the application of this repo has one functionality. It takes one sentence as input. Using Text Analysis, calculates the emotion of the sentence.

From the technical perspective, the application consists of three microservices. Each has one specific functionality:

* SA-Frontend: a Nginx web server that serves our ReactJS static files.  
* SA-WebApp: a Java Web Application that handles requests from the frontend.  
* SA-Logic: a python application that performs Sentiment Analysis.  

The repo consists of 4 parts: source code of these three microservices, and some helpful YAML files to build the deployments and services for Kubernetes cluster.  

Further instructions can be found in this article: [Learn Kubernetes in Under 3 Hours: A Detailed Guide to Orchestrating Containers](https://www.freecodecamp.org/news/learn-kubernetes-in-under-3-hours-a-detailed-guide-to-orchestrating-containers-114ff420e882)  

---

## Environment

I worked on the virutal machine of Linux Ubuntu22 on my MacOS12.2.1 M1 macbook pro. We also need to have NPM, NodeJS, JDK8, Maven, Python3, pip, git, Docker ready in our local/virtual machine we are using.  

---

## Create Images

To get everything working on the cloud, we need to build/update a container image for each service locally and push it to Docker Hub. We can also find the `Dockerfile` under each microservice's directory.  

The [guidelines](https://www.freecodecamp.org/news/learn-kubernetes-in-under-3-hours-a-detailed-guide-to-orchestrating-containers-114ff420e882) we mentioned before already has a lot specific instructions upon how to do it. Basically we need to:  
1. Build the React application for Frontend, package the Java applicaiton into a Jar for WebApp, install Python dependencies for Logic.
2. Make sure the application runs perfectly on locally on local/virtual machines (if using [Nginx](https://www.nginx.com/resources/wiki/start/topics/tutorials/install/) for Frontend on Linux etc. systems, be careful with port 80 it would be using, since it might occupy the port 80, which is also needed for our frontend container later.)
3. Use Docker Build to create images for each Microservice, and push the images up to our Docker Hub, be careful with the docker hub ID and the labels (latest, latest-amd64 etc.).

Make sure we login into our docker account before building any images.  
```
docker login -u="$DOCKER_USERNAME" -p="$DOCKER_PASSWORD"
```

***Be careful!!!*** with the Operating System we use. GKE could only support AMD64 platform (Windows), not MacOS M1/2 or Linux. And Docker in fact detects the Apple M1 Pro platform as linux/arm64/v8 too. To make images that are built on MacOS and Linux platform, we need to specify the platform to both the build command and version tag ([reference](https://stackoverflow.com/questions/42494853/standard-init-linux-go178-exec-user-process-caused-exec-format-error)).  

That saying, we can not just simply use (provided by the guidelines and the repo README files) :  
```
docker build -f Dockerfile -t $DOCKER_USER_ID/sentiment-analysis-frontend .
```
Instead, we should use to make sure the docker image is 'built' on AMD64 platform, and tag them as version ```latest-amd64``` for later use:  
```
# Build for AMD64
docker build --platform=linux/amd64 -t $DOCKER_USER_ID/sentiment-analysis-frontend:latest-amd64 .
```

Make sure to push them to the docker hub too.  
```
docker push $DOCKER_USER_ID/sentiment-analysis-frontend:latest-amd64
```

This applys for every microservice docker image.  

Specially, in the microservice ```sa-logic/``` directory we might have ```exec /bin/sh: exec format error``` problem while building image with tag ``` --platform=linux/amd64 ```. This type of error is typically seen when trying to run an executable that's incompatible with the host's architecture. In this case, it's because my Ubuntu host is running on an aarch64 architecture (also known as ARM64), but we want to build the image based on ```linux/amd64 ```. Here is how to solve it:  

1. Install Docker Buildx:
`buildx` is an experimental feature and comes bundled with Docker 19.03 and later. We need to enable experimental CLI features for it to work.

First, check if `buildx` is available:
```bash
docker buildx version
```

If it's not available or we get an error, ensure we have Docker 19.03 or later installed and that experimental CLI features are enabled.

3. Create a New Builder Instance:

```bash
docker buildx create --use --name mybuilder
```

This will create a new builder instance with the name `mybuilder` and set it as the default.

4. Bootstrap the Builder:

```bash
docker buildx inspect mybuilder --bootstrap
```

This command ensures the builder is up and running.

5. Build for amd64:

Now, use `buildx` to build the image for `amd64`:

```bash
docker buildx build --platform linux/amd64 -f Dockerfile -t $DOCKER_USER_ID/sentiment-analysis-logic:latest-amd64 . --load
```

The `--load` flag ensures that the built image is loaded back into Docker's local image store. This means we can run and inspect the image on our host. Note, however, that we won't be able to run this `amd64` image on our `aarch64` host without additional emulation (like QEMU).

Remember, even though we're building for `amd64`, we're doing so on an `aarch64` host, which is why we use QEMU and `buildx` for cross-compilation.

---

## Immigrate Images to Cloud

Now that we have our images on Docker Hud, we need to immigrate them to Cloud for future deployments.  

First, login into [Google Cloud Platform](https://console.cloud.google.com/) and open [Kubernete Engine](https://console.cloud.google.com/kubernetes/).  

Make sure you have one project and your Kubernete cluster to use, if not, create one as follow: choose standard cluster template, name your cluster, choose a non-busy zone, then all default settings.  
<p align="center">
<img width="568" alt="image" src="https://github.com/Cloud-Infrastructure-Fall-2023/hw-3-microservice-orchestration-zelinewang/assets/89945709/53526b5f-aa3f-4516-8000-cc0378c63679">
<img width="549" alt="image" src="https://github.com/Cloud-Infrastructure-Fall-2023/hw-3-microservice-orchestration-zelinewang/assets/89945709/926ff08b-e937-419c-862f-39db292c9994">
</p>

Then, open Cloud shell and choose your project, and navigate to the cluster we are working on (*click on ***CONNECT*** in your cluster page*) , authorize Cloud shell.  

Apply following commands for each microservice image to immigrate the images in your Docker Hub to Google Cloud Platform's Container Registry.  
```
docker pull $DOCKER_USER_ID/sentiment-analysis-logic:latest-amd64
docker tag $DOCKER_USER_ID/sentiment-analysis-logic:latest-amd64 gcr.io/$YOUR_PROJECT_ID/$DOCKER_USER_ID/sentiment-analysis-logic:latest-amd64
docker push gcr.io/$YOUR_PROJECT_ID/$DOCKER_USER_ID/sentiment-analysis-logic:latest-amd64
```

Now we can find all the images we need to deploy in Container Registry under ```gcr.io/$YOUR_PROJECT_ID```.  

---

## Make the deployment

This is generally how Kubernete cluster works to manage all worker nodes. There are deployment for each microservice inside of a work node, and inside of those deployments, there are many pods where the containers are running inside.  

<p align="center">
<img width="775" alt="image" src="https://github.com/Cloud-Infrastructure-Fall-2023/hw-3-microservice-orchestration-zelinewang/assets/89945709/2d205f77-53e8-43d1-a853-50b2cdb52658">
</p>

We can find some YAML files that are used in the guidelines to deploy microservices to local `Minikube` cluster and also configure the deployments, we can use them to learn how to make the deployment on cloud.  


Look into the YAML files under `k8s-mastery/resource-manifests` directory. We can find out Deployment and Service YAML files that indicates the basic configuration rules for deploying images and services:  
1. Name each microservice their names as "sa-frontend", "sa-web-app" and "sa-logic" to identify and label them and their resources.  
2. Custimize the image we want to use in pods' container.  
3. Custimize the number of pods we want in deployment.
4. Get to know what's the container port each service needs to be accessed, from `containerPort: 8080`. We know from the deployment YAML files that `sa-web-app` uses `8080`, `sa-logic` uses `5000`, `sa-frontend` uses as `80`.  
5. Know that the deployment is using rolling update strategy to update the pods.

Those are mainly from the deployment YAML. In the service YAML, we could also tell that:  
1. We use the LoadBalancer Service to keep the workload between the pods inside of the same deployment balanced.
2. Services have a different name than "sa-frontend", "sa-web-app" and "sa-logic", more like "sa-frontend-service", "sa-web-app-service" and "sa-logic-service".  
3. From the `port`, we found out there are **two ports**, **`port: 80`** and **`targetPort: 80`**, both using TCP. `port: 80` is known as the Service Port for the service to internally communicate with each other in a cluster, and for others to get access to this service. We as users can also use this port to get access to the service. For example, as users, we need to get access to `sa-frontend` to actually interact with the application at the front end, then we need to go to the Service port `80` of the `sa-frontend`. Locally, we just need to go to `localhost:80` address, the `localhost` is our local IP.  
4. `targetPort: 80` is the port on which the pod is listening, means that when someone (users/other pods, etc.) hits the Service Port, it should be transfered to the Target Port where the pod actually lays, and able to run the service.

#### Now deploy to the cluster

Make the configuration of the deployments on the cluster: 
1. Choose the `latest-amd64` images we just created and pushed to cloud Container Registry.  
2. Name the deployments as mentioned before.  
3. We can customize the `replica` number as default `3`.
4. Make sure to configure the `EXPOSE`, use LoadBalancer Service, and `port` number is all `80`, `target port` number is the same as each service `containerPort` number, which is `sa-web-app` uses `8080`, `sa-logic` uses `5000`, `sa-frontend` uses as `80`, all using TCP.
5. Wait for all the deployments are done.  

---
## Port Numbers

We might need to elaborate more on port numbers here.  

#### Port (Service Port):

* This is the port on which the service is exposed and is accessible within the cluster.  
* Other pods within the cluster will use this port to communicate with the service.  
* When you set up a service to expose a deployment, the "port" is the one you'd use to access the service from within the cluster.  

#### Target Port:

* This is the port on which the pod (or the actual application inside the pod) is listening.
* When traffic hits the service on the "port", it is forwarded to the "target port" on one of the pods that backs the service.
* The "target port" can be the same as the "port" or different, depending on the configuration.

#### Simple analogy:
Imagine a company with a main customer service phone number (the "port"). When you call that number, your call might be forwarded to any one of the individual employee's desk phones (the "target port"). The main number is what you, as an outsider, know and use. The individual desk phone numbers are internal details.

This is pretty important. Because we can get access to `sa-frontend` with `localhost:80` with a local running service, but what if the service is running on Cloud? We probably need the cloud deployed service's _**external/public IP**_ with the _**Service Port**_, which is known as the **_`External Endpoint`_** of a service.  

**Question: Why do users access the service using the "port" and not the "target port"?** The reason is abstraction and decoupling. The service acts as an abstraction layer in front of pods. Users or clients shouldn't need to know about the individual pods or their specific configurations. They only need to know about the service. This abstraction allows for flexibility:  
* Pods can be added or removed without affecting the service.  
* The actual application inside the pod can change its listening port without affecting clients; only the service's `target port` configuration would need to be updated. This means that assume we are using an external endport and using this service, now, if the company wants to change the port number of this running pod, the company doesn't have to make us change the port we are accessing but only need to change the target port that the traffic should be forwarded to.
* **It provides a consistent endpoint for clients, even as backend details change**.

---

## Changes for Cloud Deployments

Now, we noticed that, we already deployed everything to the cloud right now, but when we go to the expose endpoint of `sa-frontend` service, the service is not really working the way we expected. Why?  

To inspect the problem, we can use the `inspect` in browser to check the `Network` traffic to see if there is any problem. Go to the `indicator` under the name `sentiment`, we found there is a problem with the `App.js`, click on it and navigate to the `src/App.js` in `sa-frontend directory`. I think we found out what is the problem.  

There is also some related instructions in that guideline page too. The problem was as we deploy every service to the cloud, but we **ignore how each service should be connected and communicate with each other**. 

In local implementation on only containers, we only used `localhost:8080`, `localhost:5000` to get access to everything, for example, in `sa-frontend/src/App.js`, we use `localhost:8080` to get the sentiment passed into `sa-web-app` by it's container port.  
```
    analyzeSentence() {
        fetch('http://localhost:8080/sentiment', {
      ............
    }
```
In `sa-web-app/Dockerfile`, it sets the Environment varible SA_LOGIC_API_URL to `localhost:5000` to use it as the parameter in `java -jar sentiment-analysis-web-0.0.1-SNAPSHOT.jar --sa.logic.api.url=http://localhost:5000` to get the web-app running and get access to the `sa-logic` analysis.  
```
# Environment Variable that defines the endpoint of sentiment-analysis python api.
ENV SA_LOGIC_API_URL http://localhost:5000
```

But that's what happened in local, that's the reason we used `localhost`, in cloud, we need to specific URL to for these services to get access to each other or commnicate.  

#### Modify the source code

To achieve that, we need to modify the source, basically we need to change all the `localhost` URL into the real Cloud Service Endpoint URL to get access to services on cloud. Endpoint URL can be found in "Service and Ingress" of GKE. Since in the port number section, we already talked about we only needs `Service Port` but not `Target Port`, so there is their service port number in the Endpoint URL, but not target port number.  

* Change `localhost:8080` in `sa-frontend/src/App.js` into `35.196.48.152:80`, and `localhost:5000` in `sa-web-app/Dockerfile` into `35.190.130.116:80`. 


<p align="center">
<img width="928" alt="image" src="https://github.com/Cloud-Infrastructure-Fall-2023/hw-3-microservice-orchestration-zelinewang/assets/89945709/e571e221-e803-40f7-a92e-f259c1b2b7d4">
</p>

---

## Rebuild the images and Modify the Deployment YAML file

After modifying the source code, we need to build the images again, based on our changed source code (make sure to run `npm run build` before building the image for `sa-frontend`).  

Then push to docker hub again, and push to GCR google container registry again, with similar operations.  

Now we have updated images on cloud.  

But our deployments are already running, if we made new deployments, there will be new service endpoints too, means we have to modify the source code again? But then there is a no-end loop! 

Wait, we actually know how to solve this. Remember in the YAML file when we made the original configuration of the deployments? Now navigate to the deployment workload section, we found that we can just **modify/edit the YAML file to configure the deployment, even after the deployment is created**! How cool is that! 

<p align="center">
<img width="1436" alt="image" src="https://github.com/Cloud-Infrastructure-Fall-2023/hw-3-microservice-orchestration-zelinewang/assets/89945709/602fa4e9-404a-43fc-acfb-66ecab1bcef2">
</p>

#### Modify YAML files

Modify the YAML files of each deployment (under "Workload" section of GKE) that needs to update its image to the new images, which are built on modified source code. The place should be modified is the `image` under the `spec - container`, there is an unique image ID from the container registry, indicating which image the deployment would use. (which imagefrom different labels, different versions)  

<p align="center">
<img width="1178" alt="image" src="https://github.com/Cloud-Infrastructure-Fall-2023/hw-3-microservice-orchestration-zelinewang/assets/89945709/9d9057c5-0585-44ec-bf70-7c02bcc8ca7d">
</p>

We just need to copy the image ID of updated images from GCR (container registry), and replace the image ID of the old ones. And wait for the deployment to update for about few minutes, this might take longer than expected.  

<p align="center">
<img width="1169" alt="image" src="https://github.com/Cloud-Infrastructure-Fall-2023/hw-3-microservice-orchestration-zelinewang/assets/89945709/25cf9555-f937-4571-bd14-a1999a29721c">
</p>

Now go back to inspecting the frontend page again, we can see everything works just fine.   

<p align="center">
<img width="1436" alt="image" src="https://github.com/Cloud-Infrastructure-Fall-2023/hw-3-microservice-orchestration-zelinewang/assets/89945709/5cf800bd-95b6-4661-9b26-30eed19072fe">

<img width="1438" alt="image" src="https://github.com/Cloud-Infrastructure-Fall-2023/hw-3-microservice-orchestration-zelinewang/assets/89945709/fd8d79ad-75a3-4adf-b3bc-295964bdc967">
</p>

## Conclusion

We successfully deployed and orchestrated this microservice-based application on cloud.  

You can get access to it by its **frontend endpoint**: http://34.138.150.173:80/  

## Reference

* Open Source Git Repo: https://github.com/rinormaloku/k8s-mastery/tree/master
* [Learn Kubernetes in Under 3 Hours: A Detailed Guide to Orchestrating Containers](https://www.freecodecamp.org/news/learn-kubernetes-in-under-3-hours-a-detailed-guide-to-orchestrating-containers-114ff420e882) by Rinor Maloku
* [Stack over flow: Standard_init_linux.go:178: exec user process caused "exec format error"](https://stackoverflow.com/questions/42494853/standard-init-linux-go178-exec-user-process-caused-exec-format-error)
* [ChatGPT](https://chat.openai.com/) for only creating my table of contents.

## README of the Original Repo

This repository contains the source files needed to follow the series [Kubernetes and everything else](https://rinormaloku.com/series/kubernetes-and-everything-else/) or summarized as an article in [Learn Kubernetes in Under 3 Hours: A Detailed Guide to Orchestrating Containers](https://medium.freecodecamp.org/learn-kubernetes-in-under-3-hours-a-detailed-guide-to-orchestrating-containers-114ff420e882)  

To learn more about Kubernetes and other related topics check the following examples with the **Sentiment Analysis** application:  

* [Kubernetes Volumes in Practice](https://rinormaloku.com/kubernetes-volumes-in-practice/):
* [Ingress Controller - simplified routing in Kubernetes](https://www.orange-networks.com/blogs/210-ingress-controller-simplified-routing-in-kubernetes)
* [Docker Compose in Practice](https://github.com/rinormaloku/k8s-mastery/tree/docker-compose)
* [Istio around everything else series](https://rinormaloku.com/series/istio-around-everything-else/)
* [Simple CI/CD for Kubernetes with Azure DevOps](https://www.orange-networks.com/blogs/224-azure-devops-ci-cd-pipeline-to-deploy-to-kubernetes)
* Envoy series - to be added!
