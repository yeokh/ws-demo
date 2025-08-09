### **Quick Start OpenShift Demo with n8n** \- [https://n8n.io/](https://n8n.io/)

n8n is an AI workflow automation builder.  Using drag-n-drop UI, n8n enables you to build and implement multi-step AI agents with integrations to other tools.

![][image1]

**Tools calling with n8n LangChain agent** 

Under the hood, n8n uses LangChain, where a tool is an association between a function and its schema.  To use a tool in an n8n workflow, we need to bind/connect the tool to the model that supports tool/function calling. This gives the model awareness of the tool and the associated input schema required by the tool.  

The model will decide, when appropriate, to call the tool based on the chat message received, and will ensure it conforms to the tool's input schema in order to properly execute the corresponding function call.

Documentation:   
[https://docs.n8n.io/hosting/configuration/configuration-examples/isolation/](https://docs.n8n.io/hosting/configuration/configuration-examples/isolation/)

### **Deploying n8n on OpenShift**

Get a free OpenShift sandbox environment if you don’t already have one:   
Register for a Red Hat Developer account: https://developers.redhat.com/register  
Access your Red Hat Developer Sandbox environment: https://sandbox.redhat.com/  or  https://developers.redhat.com/developer-sandbox

Log in to your OpenShift Sandbox environment to deploy 

**Modify n8n container image file** 

Modify the n8n container file to add the commands to grant the correct file permissions for the application to run under OpenShift's strict security model.

When OpenShift starts the container, it starts as a random high-number user, e.g., user 1001050000\.  And your application, now running as user 1001050000, tries to write a file to a directory owned by root or a specific user id, your application will encounter denied permission errors. 

Add the command chmod \-R g=u in the container file to set the group's permissions to be identical to the user's (owner's) permissions.

You can use the following customized image: quay.io/kenghua\_yeo/yn8n:v1.3

**Create Custom Image for OpenShift**

Git clone and copy from n8n git repository the the custom docker/container file to the root folder \- [https://github.com/n8n-io/n8n/blob/master/docker/images/n8n-custom/Dockerfile](https://github.com/n8n-io/n8n/blob/master/docker/images/n8n-custom/Dockerfile)

In the root folder of the cloned n8n git repo, edit the docker file to change the file/folder permissions:

$ git clone https://github.com/n8n-io/n8n.git 
$ cd n8n 
$ cp docker/images/n8n-custom/Dockerfile \<customized\_container\_file\> 
$ vi \<customized\_container\_file\> 
...         
mkdir .n8n && \\         
chown node:node .n8n \\         \#\#\# Change file and folder permissions \#\#\#         && chmod \-R g=u .n8n \\         && mkdir /.n8n \\         && chmod \-R g=u /.n8n \\         && mkdir /.cache \\         && chmod \-R g=u /.cache ENV SHELL /bin/sh ... $ podman build \--no-cache \-f \<customized\_container\_file\> \-t quay.io/\<customized\_n8n\_image\_repo\>:v1.0 .  |
| :---- |

To prevent cached layers being used during a build, use the \--no-cache option with the podman build command. This ensures that all layers are rebuilt from scratch, avoiding any cached results. Do this if your build does not produce the expected modified image.

Test the image by running it locally:

| $ podman  volume create n8n\_data $ podman run \-it \--rm \--name n8n \-e N8N\_RUNNERS\_ENABLED='true' \-e N8N\_SECURE\_COOKIE='false' \-p 5678:5678 \-v n8n\_data:/home/node/.n8n  quay.io/\<customized\_n8n\_image\_repo\>:v1.0 \#\# You can also try running the unmodified n8n container image 
$ podman run \-it \--rm \--name n8n \-e N8N\_RUNNERS\_ENABLED='true' \-e N8N\_SECURE\_COOKIE='false' \-p 5678:5678 \-v n8n\_data:/home/node/.n8n  docker.n8n.io/n8nio/n8n |
| :---- |

To deploy to an OpenShift cluster, push the image to an external image registry that is accessible from the OpenShift cluster.  You may need to log in to the external image registry, for example quay.io, and set the repository to public for testing on OpenShift sandbox environment.

| $ podman login \<your image repo \-u ID and \-p Credential\> quay.io $ podman push quay.io/\<customized\_n8n\_image\_repo\>:v1.0 |
| :---- |

### **Deploy on OpenShift**

| \#\# login to OpenShift workshop project/namespace $ oc login  $ oc project \#\# Deploy the custom n8n container  $ oc apply \-f n8n-deploy.yaml $ oc get pods $ oc events pod/xxx $ oc logs xxx $ oc apply \-f n8n-service.yaml $ oc get services $ oc apply \-f n8n-route.yaml $ oc get routes \#\# Delete the deployed resources $ oc delete \-f n8n-route.yaml $ oc delete \-f n8n-service.yaml $ oc delete \-f n8n-deploy.yaml |
| :---- |

### **Working Examples**

Here are the example yaml files used:

| $ vi n8n-deploy.yaml \--- apiVersion: apps/v1 kind: Deployment metadata:   labels:     service: n8n   name: n8n spec:   replicas: 1   selector:     matchLabels:       service: n8n   strategy:     type: Recreate   template:     metadata:       labels:         service: n8n     spec:       containers:         \- command:             \- /bin/sh           args:             \- \-c             \- sleep 5; n8n start           env:             \- name: N8N\_PROTOCOL               value: http             \- name: N8N\_PORT               value: "5678"             \- name: N8N\_SECURE\_COOKIE               value: "false"           image: quay.io/kenghua\_yeo/yn8n:v1.3           name: n8n           ports:             \- containerPort: 5678           resources:             requests:               memory: "250Mi"             limits:               memory: "500Mi"       restartPolicy: Always  |
| :---- |

| $ vi n8n-service.yaml \--- apiVersion: v1 kind: Service metadata:   labels:     service: n8n   name: n8n spec:   type: ClusterIP   ports:     \- name: "5678"       port: 5678       targetPort: 5678       protocol: TCP   selector:     service: n8n  |
| :---- |

To ensure an external route is created with HTTPS, create the route specification with TLS termination:

| $ vi n8n-route.yaml \--- kind: Route apiVersion: route.openshift.io/v1 metadata:   name: n8n   labels:     service: n8n spec:   to:     kind: Service     name: n8n     weight: 100   port:     targetPort: 5678   tls:     termination: edge   wildcardPolicy: None |
| :---- |

To test the n8n HTTP request tool, you can use the following URLs to get SG weather data: https://api-open.data.gov.sg/v2/real-time/api/twenty-four-hr-forecast
