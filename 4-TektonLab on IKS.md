

# TEKTON PIPELINES LAB on IKS

![image-20200314031223750](/Users/phil/Library/Application Support/typora-user-images/image-20200314031223750.png)



Container-based software development is growing. Since it’s easy to replicate the environment, developers generally create applications on their desktop, and debug and test them locally. Later, they build and deploy the application to a Kubernetes cluster.

In this tutorial, I show you two ways to deploy an application to a Kubernetes cluster on IBM Cloud:

- Using a `kubectl` CLI without a DevOps pipeline
- Using a Tekton Pipeline (which is a Kubernetes-style continuous integration and continuous delivery (CI/CD) pipeline)

## Prerequisites

To complete this tutorial, you need to:

- Create an [IBM Cloud](https://cloud.ibm.com/login?cm_sp=ibmdev-_-developer-tutorials-_-cloudreg) account.

- Get an instance of [Kubernetes Service on IBM Cloud](https://cloud.ibm.com/kubernetes/catalog/cluster), which should take approximately 20 minutes.

- Access a Kubernetes cluster through the `kubectl` CLI. To access the instructions, go to the **IBM Cloud dashboard > [your cluster] > Access**.

- Create a namespace on the IBM Cloud container registry. To do so, go to your IBM Cloud dashboard and click **Navigation > Kubernetes > Registry > Namespaces**.

- Configure the [Git CLI](https://git-scm.com/book/en/v2/Getting-Started-Installing-Git). 


## Estimated time

After your prerequisites are configured, this tutorial takes about **40 minutes**.



## Task A - Build and deploy an application on IBM Cloud Kubernetes Service using kubectl

To build and deploy an application on a Kubernetes cluster, you generally perform the following steps:

1. Write a Dockerfile for your application and build the container image using Dockerfile.
2. Upload the built container image to the accessible container registry.
3. Create a Kubernetes deployment using the container image and deploy the application to an IBM Cloud Kubernetes Service cluster using configuration (YAML) files.

For this tutorial, we have taken a simple Hello World Node.js application to deploy on Kubernetes as shown. The following is code from our sample app; use one that you have on hand.

```javascript
  const app = require('express')()

  app.get('/', (req, res) => {
    res.send("Hello from Appsody!");
  });

  var port = 3300;

  var server = app.listen(port, function () {
    console.log("Server listening on " + port);
  })

  module.exports.app = app;
```

Create and clone a github repo:

```
cd
mkdir deploy
cd deploy
git clone https://github.com/IBM/deploy-app-using-tekton-on-kubernetes deploy-app
cd deploy-app/src
```

Check your IBM Cloud registry:

```bash
ibmcloud cr api
```

Results

```bash
# ibmcloud cr api
                           
Registry API endpoint   https://uk.icr.io/api   

OK
```

Take a note of the registry. In that case **uk.icr.io**

Build and push it to IBM Cloud Container registry where namespace is the one you created in IKS-Lab.md.

```
ibmcloud cr build -t uk.icr.io/<namespace>/builtApp:1.0 .
```

Results

```bash
# ibmcloud cr build -t uk.icr.io/ireg/hello:1.0 .
Sending build context to Docker daemon   5.12kB
Step 1/7 : FROM node:alpine
 ---> b01d82bd42de
Step 2/7 : COPY app.js /app/app.js
 ---> Using cache
 ---> 962052c0ed3b
Step 3/7 : COPY package.json /app/package.json
 ---> Using cache
 ---> 2bdbf3d2d97c
Step 4/7 : RUN cd /app && npm install
 ---> Using cache
 ---> dc1810bd90ae
Step 5/7 : ENV WEB_PORT 3300
 ---> Using cache
 ---> 92ff94d39dc1
Step 6/7 : EXPOSE  3300
 ---> Using cache
 ---> c7c9ac470b16
Step 7/7 : CMD ["node", "/app/app.js"]
 ---> Using cache
 ---> d96b8cee6467
Successfully built d96b8cee6467
Successfully tagged uk.icr.io/ireg/hello:1.0
The push refers to repository [uk.icr.io/ireg/hello]
debddb1b4cb1: Pushed 
568966cbd2fb: Pushed 
54e067222fef: Pushed 
85b69ca8f2a0: Pushed 
f9ad34829dbf: Pushed 
de5315d732c2: Pushed 
5216338b40a7: Pushed 
1.0: digest: sha256:884c4a49042459477a5d8a4620d0a6e66a1de490ab6b8795bcf4deaf994e237a size: 1783

OK
```

Verify whether the image is uploaded to the container registry

```bash
ibmcloud cr images
```

```bash
# ibmcloud cr images
uk.icr.io/ireg/hello              1.0      884c4a490424   ireg        4 minutes ago   48 MB    No Issues   
```



Update deploy target in deploy.yaml where namespace is the one you created in IKS-Lab.md

```
sed -i '' s#IMAGE#uk.icr.io/<namespace>/hello:1.0# deploy.yaml
```

Results:

```yaml
more deploy.yaml
apiVersion: v1
kind: Service
metadata:
  name: app
  labels:
    app: app
spec:
  type: NodePort
  ports:
    - port: 3300
      name: app
      nodePort: 32426
  selector:
    app: app
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app
  labels:
    app: app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: app
  template:
    metadata:
      labels:
        app: app
    spec:
      containers:
      - name: app
        image: uk.icr.io/ireg/hello:1.0
        ports:
        - containerPort: 3300
```

Run deploy configuration

```bash
kubectl create -f deploy.yaml
```

Results

```bash
# kubectl create -f deploy.yaml
service/app created
deployment.apps/app created
```

Verify output - pod and service should be up and running

```bash
# kubectl get pods
NAME                  READY   STATUS    RESTARTS   AGE
app-d78b485cb-567ql   1/1     Running   0          81s

# kubectl get service
NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
app          NodePort    172.21.43.135   <none>        3300:32426/TCP   116s
```

Retrieve the public IP of a Kubernetes cluster from your IBM Cloud dashboard

![image-20200314015332345](../../2020-02%20HybridClouds/Labs/images/image-20200314015332345-4147212.png)

In a browser use the following URL: 

```http
http://184.172.229.6:32426
```

Results:

![image-20200314015638938](../../2020-02%20HybridClouds/Labs/images/image-20200314015638938-4147399.png)





To build, test, and deploy applications faster and more reliably, you need to automate this entire workflow. Following a CI/CD methodology reduces the overhead of development and manual deployment processes, which can save you significant time and effort.

The next section of this tutorial explains the build and deploy approach using Tekton Pipelines.

## Task B - Build and deploy an application on IBM Cloud Kubernetes Service using Tekton Pipeline

[Tekton](https://github.com/tektoncd/pipeline) is a powerful and flexible Kubernetes-native open source framework for creating CI/CD systems. It allows you to build, test, and deploy across multiple cloud providers or on-premises systems by abstracting away the underlying implementation details.

Before I show you how to use Tekton Pipelines, here are some of the high-level concepts you need to understand:

The Tekton Pipeline project extends the Kubernetes API by five additional custom resource definitions (CRDs) to define pipelines:

- A *task* is an individual job and defines a set of build steps such as compiling code, running tests, and building and deploying images.
- A *Taskrun* runs the task you defined. With taskrun, it’s possible to execute a single task, which binds the inputs and outputs of the task.
- *Pipeline* describes a list of tasks that compose a pipeline.
- *Pipelinerun* defines the execution of a pipeline. It references the pipeline to run and which PipelineResource(s) to use as input and output.
- The *PipelineResource* defines an object that is an input (such as a Git repository) or an output (such as a Docker image) of the pipeline.

To automate the application’s build and deploy workflow using Tekton Pipelines, follow these steps:

### 1. Add the Tekton Pipelines component to your Kubernetes cluster

Be sure your are still connected to your IKS cluster:

```bash
# kubectl get nodes
NAME           STATUS   ROLES    AGE     VERSION
10.76.215.20   Ready    <none>   7h51m   v1.16.7+IKS
```

As a first step, **add the Tekton Pipelines to your Kubernetes** cluster using the following command:

```bash
kubectl apply --filename https://storage.googleapis.com/tekton-releases/latest/release.yaml
```

Results:

```bash
# kubectl apply --filename https://storage.googleapis.com/tekton-releases/latest/release.yaml
namespace/tekton-pipelines created
podsecuritypolicy.policy/tekton-pipelines created
clusterrole.rbac.authorization.k8s.io/tekton-pipelines-admin unchanged
serviceaccount/tekton-pipelines-controller created
clusterrolebinding.rbac.authorization.k8s.io/tekton-pipelines-controller-admin unchanged
customresourcedefinition.apiextensions.k8s.io/clustertasks.tekton.dev unchanged
customresourcedefinition.apiextensions.k8s.io/conditions.tekton.dev unchanged
customresourcedefinition.apiextensions.k8s.io/images.caching.internal.knative.dev unchanged
customresourcedefinition.apiextensions.k8s.io/pipelines.tekton.dev unchanged
customresourcedefinition.apiextensions.k8s.io/pipelineruns.tekton.dev unchanged
customresourcedefinition.apiextensions.k8s.io/pipelineresources.tekton.dev unchanged
customresourcedefinition.apiextensions.k8s.io/tasks.tekton.dev unchanged
customresourcedefinition.apiextensions.k8s.io/taskruns.tekton.dev unchanged
service/tekton-pipelines-controller created
service/tekton-pipelines-webhook created
clusterrole.rbac.authorization.k8s.io/tekton-aggregate-edit unchanged
clusterrole.rbac.authorization.k8s.io/tekton-aggregate-view unchanged
configmap/config-artifact-bucket created
configmap/config-artifact-pvc created
configmap/config-defaults created
configmap/config-logging created
configmap/config-observability created
deployment.apps/tekton-pipelines-controller created
deployment.apps/tekton-pipelines-webhook created
```

The installation creates two pods which you can check using the following command. Make sure to wait until the pods are in a running state.

```
  kubectl get pods --namespace tekton-pipelines
```

Results

```bash
# kubectl get pods --namespace tekton-pipelines
NAME                                           READY   STATUS    RESTARTS   AGE
tekton-pipelines-controller-5b75cdfb95-bh9mj   1/1     Running   0          22s
tekton-pipelines-webhook-b848dcd97-bwjjl       1/1     Running   0          15s
```



For more information on this, refer to the [Tekton documentation](https://github.com/tektoncd/pipeline/blob/master/docs/install.md#adding-the-tekton-pipelines). After completing these steps, your Kubernetes cluster is ready to run Tekton Pipelines. Let’s start by creating the definition of custom resources.



### 2. Create a PipelineResource

In this tutorial’s example, the source code of the application, Dockerfile, and deployment configuration is available in the [GitHub repository](https://github.com/IBM/deploy-app-using-tekton-on-kubernetes) that you cloned earlier.

To create the input PipelineResource to access the Git repository, do the following:

In the `git.yaml` file, define the `PipelineResource` for the Git repository by doing the following:

- Specify the resource `type` as Git.

- Provide the Git repository URL as `url`.

- Provivde `revision` as the name of the branch of the Git repository to be used.


The complete YAML file is available at `~/tekton-pipeline/resources/git.yaml`. 

```bash
# cat tekton-pipeline/resources/git.yaml
apiVersion: tekton.dev/v1alpha1
kind: PipelineResource
metadata:
  name: git
spec:
  type: git
  params:
    - name: revision
      value: master
    - name: url
      value: https://github.com/IBM/deploy-app-using-tekton-on-kubernetes
```

As Tekton is using Customer Resource Definitions (CRDs), you will notice that Kind is set to PipelineResource.

Apply the file to the cluster as shown.

```
 cd ~/tekton-pipeline
 kubectl apply -f resources/git.yaml
```

Results

```bash
# kubectl apply -f resources/git.yaml
pipelineresource.tekton.dev/git created
```



### 3. Create tasks

A *task* defines the steps of the pipeline. To deploy an application to a cluster using source code in the Git repository, we define two tasks — `build-image-from-source` and `deploy-to-cluster`. In the task definition, the parameters used as arguments (`args`) are referred to as `$(inputs.params.<var_name>)`.

**Define build-image-from-source**

This task includes two <u>steps</u>:

1. The `list-src` step lists the source code from the cloned repository. This is done to verify whether source code is cloned properly.
2. The `build-and-push` step builds the container image using Dockerfile and pushes the built image to the container registry. In this example, we use Kaniko to build and push the image. You can use Kaniko, builday, podman, etc. Kaniko uses the Dockerfile name, its location, and destination to upload the container image as arguments.

You can have a look to the task definition (notice kind: Task)

```bash
# cat task/build-src-code.yaml
apiVersion: tekton.dev/v1alpha1
kind: Task
metadata:
  name: build-image-from-source
spec:
  inputs:
    resources:
      - name: git-source
        type: git
    params:
      - name: pathToContext
        description: The path to the build context, used by Kaniko - within the workspace
        default: .
      - name: pathToDockerfile
        description: The path to the dockerfile to build
        default: Dockerfile
      - name: imageUrl
        description: value should be like - us.icr.io/test_namespace/builtImageApp
      - name: imageTag
        description: Tag to apply to the built image
  steps:
    - name: list-src
      image: alpine
      command:
        - "ls"
      args:
        - "$(inputs.resources.git-source.path)"
    - name: build-and-push
      image: gcr.io/kaniko-project/executor
      command:
        - /kaniko/executor
      args:
        - "--dockerfile=$(inputs.params.pathToDockerfile)"
        - "--destination=$(inputs.params.imageUrl):$(inputs.params.imageTag)"
        - "--context=$(inputs.resources.git-source.path)/$(inputs.params.pathToContext)/"
```



All required parameters are passed through parameters. Apply the file to the cluster using the following command:

```
  kubectl apply -f task/build-src-code.yaml
```

Results

```bash
# kubectl apply -f task/deploy-to-cluster.yaml
task.tekton.dev/deploy-application created
```



**Define deploy-to-cluster**

Now let’s deploy the application in a pod using the built container image, and make it available as a service to access from anywhere. This task uses the deployment configuration located as `~/src/deploy.yaml`.

This task includes two <u>steps</u>:

1. The `update-yaml` step updates the container image URL in place of `IMAGE` in the deploy.yaml.
2. The `deploy-app` step deploys the application in a Kubernetes pod and exposes it as a service using `~/src/deploy.yaml`. This step uses `kubectl` to create a deployment configuration on a Kubernetes cluster.

You can have a look to the task definition (notice kind: Task).

```bash
# cat task/deploy-to-cluster.yaml
apiVersion: tekton.dev/v1alpha1
kind: Task
metadata:
  name: deploy-application
spec:
  inputs:
    resources:
      - name: git-source
        type: git
    params:
      - name: pathToContext
        description: The path to the build context, used by Kaniko - within the workspace
        default: .
      - name: pathToYamlFile
        description: The path to the yaml file to deploy within the git source
        default: deploy.yaml
      - name: imageUrl
        description: Url of image repository
        default: url
      - name: imageTag
        description: Tag of the images to be used.
        default: "latest"
  steps:
    - name: update-yaml
      image: alpine
      command: ["sed"]
      args:
        - "-i"
        - "-e"
        - "s;IMAGE;$(inputs.params.imageUrl):$(inputs.params.imageTag);g"
        - "$(inputs.resources.git-source.path)/$(inputs.params.pathToContext)/$(inputs.params.pathToYamlFile)"
    - name: deploy-app
      image: lachlanevenson/k8s-kubectl
      command: ["kubectl"]
      args:
        - "apply"
        - "-f"
        - "$(inputs.resources.git-source.path)/$(inputs.params.pathToContext)/$(inputs.params.pathToYamlFile)"

(END)      - name: pathToContext
        description: The path to the build context, used by Kaniko - within the workspace
        default: .
      - name: pathToYamlFile
        description: The path to the yaml file to deploy within the git source
        default: deploy.yaml
      - name: imageUrl
        description: Url of image repository
        default: url
      - name: imageTag
        description: Tag of the images to be used.
        default: "latest"
  steps:
    - name: update-yaml
      image: alpine
      command: ["sed"]
      args:
        - "-i"
        - "-e"
        - "s;IMAGE;$(inputs.params.imageUrl):$(inputs.params.imageTag);g"
        - "$(inputs.resources.git-source.path)/$(inputs.params.pathToContext)/$(inputs.params.pathToYamlFile)"
    - name: deploy-app
      image: lachlanevenson/k8s-kubectl
      command: ["kubectl"]
      args:
        - "apply"
        - "-f"
        - "$(inputs.resources.git-source.path)/$(inputs.params.pathToContext)/$(inputs.params.pathToYamlFile)"

~
(END)      - name: pathToContext
        description: The path to the build context, used by Kaniko - within the workspace
        default: .
      - name: pathToYamlFile
        description: The path to the yaml file to deploy within the git source
        default: deploy.yaml
      - name: imageUrl
        description: Url of image repository
        default: url
      - name: imageTag
        description: Tag of the images to be used.
        default: "latest"
  steps:
    - name: update-yaml
      image: alpine
      command: ["sed"]
      args:
        - "-i"
        - "-e"
        - "s;IMAGE;$(inputs.params.imageUrl):$(inputs.params.imageTag);g"
        - "$(inputs.resources.git-source.path)/$(inputs.params.pathToContext)/$(inputs.params.pathToYamlFile)"
    - name: deploy-app
      image: lachlanevenson/k8s-kubectl
      command: ["kubectl"]
      args:
        - "apply"
        - "-f"
        - "$(inputs.resources.git-source.path)/$(inputs.params.pathToContext)/$(inputs.params.pathToYamlFile)"
```

All required parameters are passed through parameters.

Apply the file to the cluster as:

```
  kubectl apply -f task/deploy-to-cluster.yaml
```

Results

```bash
# kubectl apply -f task/deploy-to-cluster.yaml
task.tekton.dev/deploy-application created
```



### 4. Create a pipeline

A pipeline lists the tasks to be executed. It provides the input, output resources, and input parameters required by each task. If there is any dependency between the tasks, that is also addressed.

In the `tekton-pipeline/resources/pipeline.yaml`:

- A pipeline uses the above mentioned tasks `build-image-from-source` and `deploy-to-cluster`.
- The `runAfter` key is used here because the tasks need to be executed one after the other.
- The PipelineResource (Git repository) is provided through the `resources` key.

You can have a look to the pipeline definition (notice kind: pipeline).

```bash
# cat pipeline/pipeline.yaml
apiVersion: tekton.dev/v1alpha1
kind: Pipeline
metadata:
  name: application-pipeline
spec:
  resources:
    - name: git-source
      type: git
  params:
    - name: pathToContext
      description: The path to the build context, used by Kaniko - within the workspace
      default: .
    - name: pathToYamlFile
      description: The path to the yaml file to deploy within the git source
      default: config.yaml
    - name: imageUrl
      description: Url of image repository
      default: deploy_target
    - name: imageTag
      description: Tag to apply to the built image
      default: latest
  tasks:
  - name: build-image-from-source
    taskRef:
      name: build-image-from-source
    params:
      - name: pathToContext
        value: "$(params.pathToContext)"
      - name: imageUrl
        value: "$(params.imageUrl)"
      - name: imageTag
        value: "$(params.imageTag)"
    resources:
      inputs:
        - name: git-source
          resource: git-source
  - name: deploy-application
    taskRef:
      name: deploy-application
    runAfter:
      - build-image-from-source
    params:
      - name: pathToContext
        value: "$(params.pathToContext)"
      - name: pathToYamlFile
        value: "$(params.pathToYamlFile)"
      - name: imageUrl
        value: "$(params.imageUrl)"
      - name: imageTag
        value: "$(params.imageTag)"
    resources:
      inputs:
        - name: git-source
          resource: git-source
```



All required parameters are passed through parameters. The parameters value is defined in the pipeline as `$(params.imageUrl)` which is different than the `args` in the task definition. Apply this configuration as:

```
  kubectl apply -f pipeline/pipeline.yaml
```

Results

```bash
# kubectl apply -f pipeline/pipeline.yaml
pipeline.tekton.dev/application-pipeline created
```

At this point, you only have set up several definitions. Nothing has been yet executed.



### 5. Create PipelineRun

To execute the pipeline, you need a `PipelineRun` resource definition, which passes all required parameters. `PipelineRun` triggers the pipeline, and the pipeline, in turn, creates `TaskRuns` and so on. In a similar manner, all parameters get substituted down to the tasks.

If a parameter is not defined in the `PipelineRun`, then the default value gets picked up from the `params` under `spec` from the resource definition itself. For example, the `pathToDockerfile` parameter is used in the task `build-image-from-source`, but its value is not provided in `pipeline-run.yaml`. Because of this, its default value defined in `~/tekton-pipeline/build-src-code.yaml` is used during the task execution.

In the `PipelineRun` definition, `tekton-pipeline/pipeline/pipeline-run.yaml`:

- References the pipeline `application-pipeline` created through `pipeline.yaml`.
- References the PipelineResource `git` to use as input.
- Provides the value of parameters under `params` which are required during the execution of the pipeline and the tasks.
- Specifies a service account.

Note that through the pipeline, you can push images to the registry and deploy it to a cluster. You need to ensure that it has the sufficient privileges to access the container registry and the cluster. The credentials for the registry are provided by a service account. You need to define a service account before executing `PipelineRun`.

*Note: Do not apply the PipelineRun file yet because you still need to define the service account for it.*

### 6. Create a service account

To access the protected resources, set up a service account which uses secrets to create or modify Kubernetes resources. The IBM Cloud Kubernetes Service is configured to use IBM Cloud Identity and Access Management (IAM) roles. These roles determine the actions that users can perform on IBM Cloud Kubernetes.

**Generate an API key**

To generate an API key using IBM Cloud Dashboard, follow the instructions in the [IBM Cloud documentation](https://cloud.ibm.com/docs/iam?topic=iam-userapikey#create_user_key). You can also use the following CLI command to create the API key:

```
  ibmcloud iam api-key-create MyKey -d "this is my API key" --file key_file.json
  cat key_file.json | grep apikey
```

Results

```bash
# ibmcloud iam api-key-create MyKey -d "this is my API key" --file key_file.json
Creating API key MyKey as thomas1@fr.ibm.com...
OK
API key MyKey was created
Successfully save API key information to key_file.json
# cat key_file.json | grep apikey
	"apikey": "5EUAUYBkJRNc3Vi5kb9R-IT3W-rFsA77Io0vIBp65zaZ"
```

Copy the `apikey`. You will use it in the next step.

**Create secrets**

To create a secret, use the following code. `APIKEY` is the one that you created and `REGISTRY` is the registry API endpoint for your cluster. An example would be `us.icr.io`.

```
  kubectl create secret generic ibm-cr-secret --type="kubernetes.io/basic-auth" --from-literal=username=iamapikey --from-literal=password=<APIKEY>
```

Results

```bash
# kubectl create secret generic ibm-cr-secret --type="kubernetes.io/basic-auth" --from-literal=username=iamapikey --from-literal=password=5EUAUYBkJRNc3Vi5kb9R-IT3W-rFsA77Io0vIBp65zaZ
secret/ibm-cr-secret created
```

Then annotate the secret:

```
kubectl annotate secret ibm-cr-secret tekton.dev/docker-0=<REGISTRY>
```

Results

``` bash
# kubectl annotate secret ibm-cr-secret tekton.dev/docker-0=uk.icr.io
secret/ibm-cr-secret annotated
```

This creates a secret named `ibm-cr-secret` which is then used in the configuration file for the service account.

In the configuration file `tekton-pipeline/pipeline/service-account.yaml`:

- The `ServiceAccount` resource uses the secret `ibm-cr-secret`.
- As per the definition of a secret resource, the newly built secret is populated with an API token for the service account.

The next step is to define roles. A role can only be used to grant access to resources within a single namespace. You must include appropriate resources and apiGroups in rules or it will fail because of access issues. A role binding grants the permissions defined in a role to a user or set of users. It holds a list of subjects (users, groups, or service accounts) and a reference to the role being granted.

```bash
# cat cat pipeline/service-account.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: service-account
secrets:
- name: ibm-cr-secret

---

apiVersion: v1
kind: Secret
metadata:
  name: kube-api-secret
  annotations:
    kubernetes.io/service-account.name: service-account
type: kubernetes.io/service-account-token

---

kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  namespace: default
  name: pipeline-role
rules:
- apiGroups: ["extensions", "apps", ""]
  resources: ["services", "deployments", "pods"]
  verbs: ["get", "create", "update", "patch", "list", "delete"]

---

apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: pipeline-role-binding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: pipeline-role
subjects:
- kind: ServiceAccount
  name: service-account
  namespace: default 
```

Apply this configuration as:

```
  kubectl apply -f pipeline/service-account.yaml
```

Results

```bash
# kubectl apply -f pipeline/service-account.yaml
serviceaccount/service-account created
```



### 7.Execute the pipeline

Before executing the `PipelineRun`, modify the `imageUrl` and the `imageTag` in `tekton-pipeline/pipeline/pipelinerun.yaml`. Refer to the Set up deploy target section above to decide on an image URL and tag. If the image URL is *us.icr.io/test_namespace/builtApp* and the image tag is *1.0*, then update the configuration file to:

```
  sed -i '' s#IMAGE_URL#us.icr.io/test_namespace/builtApp# pipeline/pipelinerun.yaml
  sed -i '' s#IMAGE_TAG#1.0# pipeline/pipelinerun.yaml
```

You can look at the file before executing the pipelinerun:

```bash
# cat pipeline/pipeline-run.yaml
apiVersion: tekton.dev/v1alpha1
kind: PipelineRun
metadata:
  name: application-pipeline-run
spec:
  pipelineRef:
    name: application-pipeline
  resources:
    - name: git-source
      resourceRef:
        name: git
  params:
    - name: pathToContext
      value: "src"
    - name: pathToYamlFile
      value: "deploy.yaml"
    - name: "imageUrl"
      value: "us.icr.io/ireg/hello"
    - name: "imageTag"
      value: "1.0"
  serviceAccountName: service-account
```



Now, create the `PipelineRun` configuration, like so:

```
  kubectl create -f pipeline/pipeline-run.yaml
```

This creates a pipeline with the below message on your terminal:

```bash
  # kubectl create -f pipeline/pipeline-run.yaml
  pipelinerun.tekton.dev/application-pipeline-run created
```

Check the status of your new pipeline:

```
  kubectl describe pipelinerun application-pipeline-run
```

You may need to rerun this command based on the status. It shows the interim status as:

```
Status:
  Conditions:
    Last Transition Time:  2019-11-11T06:51:06Z
    Message:               Not all Tasks in the Pipeline have finished executing
    Reason:                Running
    Status:                Unknown
    Type:                  Succeeded

   ...
   ...
   Events:              <none>
```

Note the message where it tells you that “Not all Tasks in the Pipeline have finished executing.”

Once the execution of your pipeline is complete, you should see the following as an output of the `describe` command. The Message now reads: “All Tasks have completed executing.”

```
Status:
  Completion Time:  2019-11-07T09:41:59Z
  Conditions:
    Last Transition Time:  2019-11-07T09:41:59Z
    Message:               All Tasks have completed executing
    Reason:                Succeeded
    Status:                True
    Type:                  Succeeded
..
..
Events:
  Type    Reason     Age   From                 Message
  ----    ------     ----  ----                 -------
  Normal  Succeeded  7s    pipeline-controller  All Tasks have completed executing
```

In case of failure, it shows which task has failed. It also gives you the additional details to check logs. For details about a resource (for instance, the pipeline) use the `kubectl describe` command to get more information.

```
  kubectl describe <resource> <resource-name>
```

### 8. Verify your results

To verify whether the pod and service is running as expected, check the output of the following commands:

```bash
  # kubectl get pods
  NAME                                                              READY   STATUS      RESTARTS   AGE
app-d78b485cb-567ql                                               1/1     Running     0          69m
application-pipeline-run-build-image-from-source-4ct9x-po-t6lxj   0/3     Completed   0          2m21s
application-pipeline-run-deploy-application-km86w-pod-kdj9j       0/3     Completed   0          100s

# kubectl get service
    NAME         TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
    app          NodePort    xxx.xx.xx.xxx   <none>        3300:32426/TCP   4m51s
```

After successful execution of `PipelineRun`, the application is accessible at `http://<public-ip-of-kubernetes-cluster>:32426/`, where you can retrieve the public IP of your Kubernetes cluster from your IBM Cloud dashboard, and the port 32426 is defined as `nodePort` in `deploy.yaml`.

Congratulations! You successfully deployed your application using a Tekton Pipeline. You should now understand the basics of Tekton Pipelines and how to get started on building your own. There are more features available, including webhooks and web-based dashboards. I suggest trying it out with IBM Cloud Kubernetes Service.

## End of Lab

Hopefully this tutorial showed you how and why you should use Tekton Pipelines for deploying an application to Kubernetes. Tekton is included in [IBM Cloud Pak for Applications](https://cloud.ibm.com/catalog/content/ibm-cp-applications?cm_sp=ibmdev-_-developer-tutorials-_-cloudreg), which offer a faster, more secure way to move your business applications to the cloud, in a container-enabled environment. Cloud Pak for Applications is built and supported on Red Hat OpenShift. Explore and try out more with the help of this [developer guide](https://developer.ibm.com/series/developers-guide-to-ibm-cloud-pak-for-applications/).





