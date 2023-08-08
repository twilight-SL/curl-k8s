# Kubernetes

> Kubernetes, also known as K8s, is an open source system for managing containerized applications across multiple hosts.
> It provides basic mechanisms for the deployment, maintenance, and scaling of applications. 
> To scrape more detailed information, please refer to the [Kubernetes GitHub](https://github.com/kubernetes/kubernetes).

### **Authorize Pod to use Kubernetes APIs**

#### **Description**  
Objective is to use `curl` command in a Pod to retrieve all Pods in the `default` namespace. This 
involves authorization and authentication issues in kubernetes, which is a relatively complex and 
complicated field.  
<br>

#### **Environment**
*  Ubuntu 22.04  
*  Minikube version v1.31.1  
*  kubectl version
    * clientVersion: v1.27.4 
    * serverVersion: v1.27.3
    * kustomizeVersion: v5.0.1  

<br>

***
#### **Concept**  
In Kubernetes, the API server is responsible for authorizing the API request. By checking all the request 
attributes and querying all the policies, it decides whether to allow or deny the access of the API request.
All permissions are turned off by default.  

Initially, in order to obtain the access permission of the resource, in this case, pods and its sub-resource pods/log,
which needs to be obtained from the API server, and a Token is required to ask the API server for the required 
information. The Token exists in the Secret, and the Secret exists in the Service Account, but how many 
permissions the service account has is defined through Role, RoleBinding, etc., resources, which is [RBAC 
Authorization](https://kubernetes.io/docs/reference/access-authn-authz/rbac/) in Kubernetes. Therefore, in our implementation, we need to create a Service Account, Role, 
and RoleBing, and of course, a Pod must be created.  
The resources and their functions are listed respectively below:
* ` Role `: defines the rules, including what operations can be performed on which restricted resources
* ` Role Binding `: this defines what can be bound to the Role, in other words, by binding the Role, 
the permissions of the Role can be executed
* ` Service Account `: where legal tokens exist, through binding roles, it can have the permissions defined in the 
rules in Role

<br>

In addition to Kubernetes-related knowledge, this implementation also involves network-related issues.  
There are several [HTTP authentication schemes](https://developer.mozilla.org/en-US/docs/Web/HTTP/Authentication),
such as Basic, Bearer, and Digest. The Basic Token, as its name implies, it is the most basic authentication 
method of HTTP, which transmitted the user_id and password in plain text. This authentication schema is usually 
used when accessing a website or domain. By comparison, Bearer Token is used to access protected resources.
This is the knowledge about the network protocol concept [Bearer Token Usage](https://blog.yorkxin.org/posts/oauth2-6-bearer-token/)
, if you want to learn more about it, please refer to Request For Comments [RFC6750](https://datatracker.ietf.org/doc/html/rfc6750).


<br>  

***
#### **Hands-on Practice**
Initially, based on the above concepts, a Service Account must be created and bound to the Role through
Role Binding to give it the authority to execute commands. Therefore, there should be at least one yaml 
file to create all the required resource.   

Following I just list the config for creating Role and Pod due to more explanation about them.
```Yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: curl-role
rules:
  - apiGroups: [""]
    resources: ["pods", "pods/log"]
    verbs: ["get", "list"]
---
apiVersion: v1
kind: Pod
metadata:
  name: curl-pod
spec:
  serviceAccountName: curl-sa
  containers:
    - name: curl-container
      image: curlimages/curl
      command: ["sleep", "infinity"]
 ```
From above yaml config, create a Role that has the authorization of the specified resource that pods is
the namespaced resource for Pod resources, and log is a subresource of pods. To allow a Role to read
pods and also access the log subresource for each of those Pods, you should configure as above.

In this example, I use the [curlimages/curl](https://hub.docker.com/r/curlimages/curl) images in the Pod, which is 
official docker image for curl - command line tool and library for transferring data with URLs and so 
that, we need not install curl in the Pod. In addition, using the command "sleep" and "infinity" to make
this Pod run forever without exit or terminate on its own.

<br>

Below is the shell script run in the Pod to retrieve all Pods in the `default` namespace
```shell
# Path to ServiceAccount token
SERVICEACCOUNT=/var/run/secrets/kubernetes.io/serviceaccount

# Read the ServiceAccount bearer token
TOKEN=$(cat ${SERVICEACCOUNT}/token)

# Reference the internal certificate authority (CA)
CACERT=${SERVICEACCOUNT}/ca.crt

# Explore the API with TOKEN
curl --cacert ${CACERT} --header "Authorization: Bearer ${TOKEN}" -X GET https://kubernetes.default.svc/api/v1/namespaces/default/pods
```

<br>

Brief output example result is as follows.
```shell
{
  "metadata": {
    "name": "purple-nginx-66f589465f-qw6lw",
    "generateName": "purple-nginx-66f589465f-",
    "namespace": "default",
    "uid": "bc10a9de-a284-4c86-8f87-4d7973e7c671",
    "resourceVersion": "1381",
    "creationTimestamp": "2023-08-03T13:47:22Z",
    "labels": {
      "app": "purple-nginx",
      "pod-template-hash": "66f589465f"
    },
    ...
    ...
  }
}
```


You could use [kubectl](https://kubernetes.io/docs/reference/kubectl/) command to check whether the output 
is correctly listing all the Pods in `default` namespace




<br>

 For more detailed authorization and authentication in Kubernetes, please refer to the 
[official document of Kubernetes](https://kubernetes.io/docs/reference/access-authn-authz/).
