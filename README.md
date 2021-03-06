# Build and deploy a Docker image on Kubernetes using Tekton Pipelines 


# Prerequisites

Tekton requires Kubernetes version 1.15 or higher.

```bash
kubectl apply --filename https://storage.googleapis.com/tekton-releases/pipeline/latest/release.yaml

```

Monitor the installation using the following command until all components show a running status:

```bash
 kubectl get pods --namespace tekton-pipelines --watch
 ```
Install Tekton CLI - https://github.com/tektoncd/cli#installing-tkn


# Clone the repository

```bash
git clone https://github.com/docloudbetter/helloworld/
```

# Create a Task to clone the Git repository

```bash
kubectl apply -f https://raw.githubusercontent.com/tektoncd/catalog/v1beta1/git/git-clone.yaml
```

# Create a Task to build an image and push it to a container registry

```bash
kubectl apply -f https://raw.githubusercontent.com/tektoncd/catalog/v1beta1/kaniko/kaniko.yaml
```

# Create a Task to deploy an image to a Kubernetes cluster

```bash
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: deploy-using-kubectl
spec:
  workspaces:
    - name: git-source
      description: The git repo
  params:
    - name: pathToYamlFile
      description: The path to the yaml file to deploy within the git source
    - name: imageUrl
      description: Image name including repository
    - name: imageTag
      description: Image tag
      default: "latest"
    - name: imageDigest
      description: Digest of the image to be used.
  steps:
    - name: update-yaml
      image: alpine
      command: ["sed"]
      args:
        - "-i"
        - "-e"
        - "s;__IMAGE__;$(params.imageUrl):$(params.imageTag);g"
        - "-e"
        - "s;__DIGEST__;$(params.imageDigest);g"
        - "$(workspaces.git-source.path)/$(params.pathToYamlFile)"
    - name: run-kubectl
      image: lachlanevenson/k8s-kubectl
      command: ["kubectl"]
      args:
        - "apply"
        - "-f"
        - "$(workspaces.git-source.path)/$(params.pathToYamlFile)"
 ```
 
 The first step runs sed in an Alpine Linux container to update the YAML file used for deployment with the image that was built by the kaniko Task. This step requires the YAML file to have two character strings, __IMAGE__ and __DIGEST__, which are substituted with parameter values.

The second step runs kubectl using Lachlan Evenson’s popular k8s-kubectl container image to apply the YAML file to the same cluster where the pipeline is running.

As was the case in the git-clone and kaniko Tasks, this Task makes use of parameters in order to make the Task as reusable as possible. It also needs the workspace to get the deployment YAML file.

```bash
kubectl apply -f tekton/tasks/deploy-using-kubectl.yaml
```
 
# Define a ServiceAccount

```bash
kubectl create secret generic pipeline-registry-secret --type="kubernetes.io/basic-auth" --from-literal=username=<username> --from-literal=password=<password>
kubectl annotate secret pipeline-registry-secret tekton.dev/docker-0=<REGISTRY>
```

## If you are using Private registry - https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/

```bash
kubectl create secret docker-registry regcred  \
  --docker-server=<REGISTRY> \
  --docker-username=<user> \
  --docker-password=<password> \
  --docker-email=<email>

kubectl annotate secret pipeline-registry-secret tekton.dev/docker-0=<REGISTRY>
```

## Now you can create the ServiceAccount using the following YAML:

```bash
apiVersion: v1
kind: ServiceAccount
metadata:
  name: pipeline-account
secrets:
- name: pipeline-registry-secret
---
apiVersion: v1
kind: Secret
metadata:
  name: kube-api-secret
  annotations:
    kubernetes.io/service-account.name: pipeline-account
type: kubernetes.io/service-account-token
---
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: pipeline-role
rules:
- apiGroups: [""]
  resources: ["services"]
  verbs: ["get", "create", "update", "patch"]
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "create", "update", "patch"]
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
  name: pipeline-account
```

This YAML creates the following Kubernetes resources:
 
A ServiceAccount named pipeline-account. The ServiceAccount references the anthony-registry-secret secret so that the pipeline can authenticate to your private container registry when it pushes and pulls a container image.
 
A secret named kube-api-secret which contains an API credential (generated by Kubernetes) for accessing the Kubernetes API. This allows the pipeline to use kubectl to talk to your cluster.
 
A Role named pipeline-role and a RoleBinding named pipeline-role-binding. This provides the resource-based access control permissions needed for this pipeline to create and modify Kubernetes resources.
 
Now apply the file to your cluster to create the ServiceAccount and related resources:

```bash
kubectl apply -f tekton/pipeline-account.yaml
```

# Create a PipelineRun

```bash
apiVersion: tekton.dev/v1beta1
kind: PipelineRun
metadata:
  generateName: helloworld-
spec:
  pipelineRef:
    name: build-and-deploy-pipeline
  params:
    - name: gitUrl
      value: https://github.com/docloudbetter/helloworld.git
    - name: gitRevision
      value: main
    - name: pathToYamlFile
      value: "kubernetes/helloworld.yaml"
    - name: imageUrl
      value: <REGISTRY>/<NAMESPACE>/helloworld
    - name: imageTag
      value: "1.0"
  serviceAccountName: pipeline-account
  workspaces:
    - name: git-source
      persistentVolumeClaim:
        claimName: helloworld-source-pvc
 ```
 
 The PipelineRun does not have a fixed name. It uses generateName to generate a name each time it is created. This is because a particular PipelineRun resource executes the pipeline only once. If you want to run the pipeline again, you cannot modify an existing PipelineRun resource to request it to rerun. You must create a new PipelineRun resource. While you could use name to assign a unique name to your PipelineRun each time you create one, it is much easier to use generateName.
 
The pipeline resource is identified under the pipelineRef key.
 
Parameters exposed by the pipeline are set to specific values, such as the Git repository to clone, the image to build, and the YAML file to deploy. This example builds a Go program that calculates an approximation of Pi. The source includes a Dockerfile which runs tests, compiles the code, and builds an image for execution.
 
 
You must edit the helloworld-pipeline-run.yaml file to substitute the values of <REGISTRY> and <NAMESPACE> with the information for your private container registry.

Before you run the pipeline for the first time, you must create the persistent volume claim for the workspace:

```bash
kubectl create -f tekton/heloworld-pipeline-pvc.yaml
```

Before continuing, check to see that the persistent volume claim is bound:

```bash
kubectl get pvc helloworld-source-pvc
```

# Run the pipeline

```bash
kubectl create -f tekton/run/helloworld-pipeline-run.yaml
```

```bash
tkn pipelinerun list

NAME               STARTED          DURATION    STATUS
helloworld-wvz55   45 minutes ago   2 minutes   Succeeded
```
```bash
tkn pipelinerun describe helloworld-wvz55

Name:              helloworld-wvz55
Namespace:         default
Pipeline Ref:      build-and-deploy-pipeline
Service Account:   pipeline-account
Timeout:           1h0m0s
Labels:
 tekton.dev/pipeline=build-and-deploy-pipeline

🌡️  Status

STARTED          DURATION    STATUS
45 minutes ago   2 minutes   Succeeded

📦 Resources

 No resources

⚓ Params

 NAME               VALUE
 ∙ gitUrl           https://github.com/docloudbetter/helloworld.git
 ∙ gitRevision      main
 ∙ pathToYamlFile   kubernetes/helloworld.yaml
 ∙ imageUrl         registry.anthonydo.com/helloworld
 ∙ imageTag         1.0

📝 Results

 No results

📂 Workspaces

 NAME           SUB PATH   WORKSPACE BINDING
 ∙ git-source   ---        PersistentVolumeClaim (claimName=picalc-source-pvc)

🗂  Taskruns

 NAME                                         TASK NAME           STARTED          DURATION     STATUS
 ∙ helloworld-wvz55-deploy-to-cluster-c2wnf   deploy-to-cluster   43 minutes ago   27 seconds   Succeeded
 ∙ helloworld-wvz55-source-to-image-7cb46     source-to-image     45 minutes ago   1 minute     Succeeded
 ∙ helloworld-wvz55-clone-repo-j978p          clone-repo          45 minutes ago   28 seconds   Succeeded
 ```
 








  
