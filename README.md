## Kubernetes - CI/CD with Tekton & Argo CD (DevOps Cloud-native CI/CD GitOps with Tekton & ArgoCD PoC)

This is a PoC to check Tekton, Argo CD and how both tools can work together following a GitOps way.

With this repo to show how a modern and cloud native CI/CD could be implemented within a Kubernetes environment. We’ll use two different tools:

- Tekton: to implement CI stages
- Argo CD: to implement CD stages (Gitops)

This repo doesn’t try to be a master class about Tekton or Argo CD. It is only intended to set an starting point to explore both tools using them directly in a cluster.

### Tekton?

Tekton is an Open Source framework to build CI/CD pipelines directly over a Kuberentes cluster. It was originally developed at Google and was known as Knative pipelines.

Tekton defines a series of Kubernetes custom resources (CRDs) extending the Kubernetes API. Sorry, what that means? Ok, if we go to the Kubernetes official page, we can read the following definition: `Kubernetes objects are persistent entities in the Kubernetes system. Kubernetes uses these entities to represent the state of your cluster.`

So, examples of Kubernetes objects are: Pod, Service, Deployment, etc. Tekton builds its own objects to Kubernetes and deploys them into the cluster. If you feel curious about custom objects, here the official documentation is and you can also check the Tekton Github to see how these objects are. For instance, [Pipeline](https://github.com/tektoncd/pipeline/blob/main/config/300-pipeline.yaml) or [Task](https://github.com/tektoncd/pipeline/blob/main/config/300-task.yaml).



### Argo CD?

Argo CD is a delivery tool (CD) built for Kubernetes, based on GitOps movement. So, what that means? Basically that Argo CD works synchronizing “Kubernetes files” between a git repository and a Kubernetes cluster. That is, if there is a change in a YAML file, Argo CD will detect that there are changes and will try to apply those changes in the cluster.

Argo CD, like Tekton, also creates its own Kubernetes custom resources that are installed into the Kubernetes cluster.


Are they ready to be adopted?
We are facing two young platforms and that may imply that there are not many examples, documentation or even maturity failures, but it’s true that both tools are called to be the standard cloud native CI/CD according to the principal cloud players.

For instance, Tekton:

- Google Cloud: https://cloud.google.com/tekton?hl=es
- Red Hat, Openshift Pipelines based on Tekton: (https://www.openshift.com/learn/topics/pipelines)
- IBM: https://www.ibm.com/cloud/blog/ibm-cloud-continuous-delivery-tekton-pipelines-tools-and-resources
- Tanzu: https://tanzu.vmware.com/developer/guides/ci-cd/tekton-what-is/
- Jenkins X: pipelines based on Tekton (https://jenkins-x.io/es/docs/concepts/jenkins-x-pipelines)


And, talking about Argo CD:

- Red Hat: https://developers.redhat.com/blog/2020/08/17/openshift-joins-the-argo-cd-community-kubecon-europe-2020/
- IBM: https://www.ibm.com/cloud/blog/simplify-and-automate-deployments-using-gitops-with-ibm-multicloud-manager-3-1-2


### CI/CD Cloud Native?

We’re talking a lot about “cloud native” associated to Tekton & Argo CD but, what do we mean by that? Both Tekton and Argo CD are installed in the Kubernetes cluster and they are based on extending Kubernetes API. Let’s see it in detail:

- Scalability: both tools are installed in the cluster and because of that, they work creating pods to perform tasks. Pods are the way in which applications can scale horizontally … so, scalabilly are guaranteed.

- Portabillity: both tools are based on extending Kubernetes API, creating new Kubernetes objects. These objects can be installed in every Kubernetes cluster.

- Reusability: the different elements within the CI/CD process use the Kubernetes objects defined by Tekton and Argo CD in the same way that you work with deployments o service objects. That means that stages, tasks or applications are YAML files that you can store in some repository and use in every cluster with Tekton and Argo CD installed. For instance, it’s possible to use artifacts from Tekton catalog or even, it’s possible to use the Openshift catalog or building a custom one.



### What are we going to build?
We are going to build a simple CI/CD process, on Kubernetes, with these stages:

<img src="poc/doc/img/pipeline-concepto.png?raw=true" width="1000">


In this pipeline, we can see two different parts:

#### CI part, implemented by Tekton and ending with a stage in which a push to a repository is done.
- Checkout: in this stage, source code repository is cloned
- Build & Test: in this stage, we use Maven to build and execute test
- Code Analisys (TODO): code is evaluated by Sonarqube
- Publish: if everything is ok, artifact is published to Nexus
- Build image: in this stage, we build the image and publish to local registry
- Push to GitOps repo: this is the final CI stage, in which Kubernetes descriptors are cloned from the GitOps repository, they are modified in order to insert commit info and then, a push action is performed to upload changes to GitOps repository

#### CD part, implemented by Argo CD, in which Argo CD detects that the repository has changed and perform the sync action against the Kubernetes cluster.
Does it mean that we can not implement the whole process using Tekton? No, it doesn’t. It’s possible to implement the whole process using Tekton but in this case, I want to show the Gitops concept.


### Hands-on!
Requirements
To execute this PoC it’s you need to have:


#### Requirenments:
A Kubernetes cluster. If you don’t have one, you can create a K3D one using the script `create-local-cluster.sh` but, obviously, you need to have installed:

- [Docker](https://docs.docker.com/engine/install/ubuntu/)
- [kubectl](https://kubernetes.io/docs/tasks/tools/#kubectl)
- [k3d](https://k3d.io/#installation)
- [tkn](https://tekton.dev/docs/cli/) (Note: Tekton CLI)
- [Skaffold](https://skaffold.dev)  (Note: Local Kubernetes Development)


#### Repository structure
I’ve used a single repo to manage the different projects. 

Basically:

- poc: this is the main folder. Here, you can find three scripts:
- create-local-cluster.sh: this script creates a local Kubernetes cluster based on K3D.
- delete-local-cluster.sh: this script removes the local cluster
- setup-poc.sh: this script installs and configure everything neccessary in the cluster (Tekton, Argo CD, Nexus, Sonar, etc…)
- resources: this the folder used to manage the two repositories (code and gitops):
- sources-repo: source code of the service used in this poc to test the CI/CD process
- gitops_repo: repository where Kubernetes files associated to the service to be deployed are


Ok, how can I execute it?


1) Fork
The first step is to fork the repo `https://github.com/adavarski/gitops-k3d-tekton-argocd` because:

You have to modify some files to add a token & You need your own repo to perform Gitops operations


2) Add Github token
It’s necessary to add a Github Personal Access Token to Tekton can perform git operations, like push to gitops repo. If you need help to create this token, you can follow these instructions: https://docs.github.com/en/github/authenticating-to-github/creating-a-personal-access-token

The token needs to be allowed with “repo” grants.

Once the token is created, you have to copy it in these files (## INSERT TOKEN HERE):
```
poc/conf/argocd/git-repository.yaml

apiVersion: v1
kind: Secret
metadata:
  annotations:
    managed-by: argocd.argoproj.io
  name: repo-gitops
  namespace: argocd
type: Opaque
stringData:
  username: adavarski
  password: ## INSERT TOKEN HERE


poc/conf/tekton/git-access/secret.yaml

apiVersion: v1
kind: Secret
metadata:
  name: git-auth
  namespace: tekton-poc
  annotations:
    tekton.dev/git-0: https://github.com
type: kubernetes.io/basic-auth
stringData:
  username: tekton
  password: ## INSERT TOKEN HERE
```

Note: In fact, for Argo CD, create secret with the token isn’t necessary because the gitops repository in Github has public access but I think it’s interesting to keep it in order to know what you need to do in case the repository be private.


3) Create Kubernetes cluster (optional)
This step is optional. If you already have a cluster, perfect, but if not, you can create a local one based on K3D, just executing the script `poc/create-local-cluster.sh`. This script creates the local cluster and configure the private image registry to manage Docker images.

4) Setup -> Run `poc/setup-poc.sh`
This step is the most important because installs and configures everything necessary in the cluster:

- Installs Tekton  & Argo CD, including secrets to access to Git repo
- Creates the volume and claim necessary to execute pipelines
- Deploys Tekton dashboard
- Deploys Sonarqube
- Deploys Nexus and configure an standard instance
- Creates the configmap associated to Maven settings.xml, ready to publish artifacts in Nexus (with user and password)
- Installs Tekton tasks and pipelines
- Git-clone (from Tekton Hub)
- Maven (from Tekton Hub)
- Buildah (from Tekton Hub)
- Prepare Image (custom task: poc/conf/tekton/tasks/prepare-image-task.yaml)
- Push to GitOps repo (custom task: poc/conf/tekton/tasks/push-to-gitops-repo.yaml)
- Installs Argo CD application, configured to check changes in gitops repository (resources/gitops_repo)
- Update Argo CD password 

Be patient. The process takes some minutes.

This message isn’t an error. It just waiting for to Nexus admin password created when the container starts. When the Nexus container starts, at some moment, it creates a file containing the default password.
```
Configuring settings.xml (MAVEN) to work with Nexus
cat: /nexus-data/admin.password: No such file or directory
command terminated with exit code 1
```

5) Explore and play

Once everything is installed, you can play with this project:

#### Tekton Part
Tekton dashboard could be exposed locally using this command: `kubectl proxy --port=8080`

but we will use ingress -> just open this url in the browser: http://tekton.192.168.1.99.nip.io:8888

By that link you’ll access to PipelineRuns options and you’ll see a pipeline executing.

<img src="poc/doc/img/gitops-tekton-pipeline.png?raw=true" width="1000">

If there is some error we can redeploy/rerun tekton pipeline and tasks:

```
  kubectl delete -f conf/tekton/git-access -n cicd
  kubectl delete -f conf/tekton/tasks -n cicd
  kubectl delete -f conf/tekton/pipelines -n cicd

  kubectl apply -f conf/tekton/git-access -n cicd
  kubectl apply -f conf/tekton/tasks -n cicd
  kubectl apply -f conf/tekton/pipelines -n cicd

  ### use `tkn` (Tekton CLI) to list/check/etc.
  tkn pipeline list -n cicd
  tkn taskrun list -n cicd
  tkn task list -n cicd
  tkn pipeline logs -n cicd

```

If you want to check what Tasks are installed in the cluster, you can navigate to Tasks option.

<img src="poc/doc/img/gitops-tekton-tasks.png?raw=true" width="1000">

If you click in this pipelinerun you’ll see the different executed stages:

<img src="poc/doc/img/gitops-k3d-tekton-argo-tekton.png?raw=true" width="1000">

Each stage is executed by a pod. For instance, you can execute:

```
kubectl get pods -n cicd -l "tekton.dev/pipelineRun=products-ci-pipelinerun"

$ kubectl get pods -n cicd -l "tekton.dev/pipelineRun=products-ci-pipelinerun"
NAME                                              READY   STATUS      RESTARTS   AGE
products-ci-pipelinerun-checkout-pod              0/1     Completed   0          16m
products-ci-pipelinerun-publish-pod               0/2     Completed   0          16m
products-ci-pipelinerun-build-and-test-pod        0/2     Completed   0          15m
products-ci-pipelinerun-prepare-image-pod         0/1     Completed   0          15m
products-ci-pipelinerun-build-image-pod           0/3     Completed   0          13m
products-ci-pipelinerun-push-changes-gitops-pod   0/1     Completed   0          11m
 
```
 
to see how different pods are created to execute different stages:


It’s possible to access to Sonarqube to check quality issues, opening this url in the browser 

```
### Ingress create:
kubectl apply -f ingress-sonarqube.yaml -n cicd
```
Browser: http://sonarqube.192.168.1.99.nip.io:8888 (admin:admin)

<img src="poc/doc/img/sonarqube.png?raw=true" width="1000">

In this pipeline, it doesn’t check if quality gate is passed.

And It’s also possible to access to Nexus to check how the artifact has been published

```
### Ingress create:
kubectl apply -f ingress-nexus.yaml -n cicd
```

Browser: http://nexus.192.168.1.99.nip.io:8888 (admin/admin123)


<img src="poc/doc/img/nexus.png?raw=true" width="1000">

Note: All ingresses
```
$ kubectl get ing --all-namespaces
NAMESPACE          NAME                CLASS   HOSTS                           ADDRESS         PORTS   AGE
tekton-pipelines   tekton-ingress      nginx   tekton.192.168.1.99.nip.io      192.168.128.2   80      88m
argocd             argocd-ingress      nginx   argocd.192.168.1.99.nip.io      192.168.128.2   80      88m
cicd               sonarqube-ingress   nginx   sonarqube.192.168.1.99.nip.io   192.168.128.2   80      7m
cicd               nexus-ingress       nginx   nexus.192.168.1.99.nip.io       192.168.128.2   80      16s
```

As we said before, the last stage in CI part consist on performing a push action to GitOps repository. In this stage, content from GitOps repo is cloned, commit information is updated in cloned files (Kubernentes descriptors) and a push is done. The following picture shows an example of this changes:

<img src="poc/doc/img/gitops-tekton-update-infra-repo.png?raw=true" width="1000">


####  Argo CD Part

To access to Argo CD dashboard you need to perform a port-forward:

```
kubectl port-forward svc/argocd-server -n argocd 9080:443
```
Then, just open this url in the browser: https://localhost:9080/ 

But again we will use ingress URL: 

Just open http://argocd.192.168.1.99.nip.io:8888

I’ve set the admin user with these credentials: admin / `$ kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d`

In this dashboard you should be the “product service” application that manages synchronization between Kubernetes cluster and GitOps repository


<img src="poc/doc/img/gitops-k3d-tekton-argo-argo.png?raw=true" width="1000">


This application is “healthy” but as the objects associated with Product Service (Pods, Services, Deployment,…etc) aren’t still deployed to the Kubernetes cluster, you’ll find a “unknown” sync status.

Once the “pipelinerun” ends and changes are pushed to GitOps repository, Argo CD compares content deployed in the Kubernetes cluster (associated to Products Service) with content pushed to the GitOps repository and synchronize Kubernetes cluster against the repository:


Finally, the sync status become “Synced”:


6) Delete the local cluster (optional )
If you create a local cluster in step 3, there is an script to remove the local cluster. This script is `poc/delete-local-cluster.sh`


### REF (example): https://github.com/adavarski/homelab -> We can add additional system (grafana/prometheus/etc.) Apps & ApplicationSet via ArgoCD manifests (bootstrap root)
```
$ git clone https://github.com/adavarski/homelab
$ cd homelab/bootstrap/root/
$ ./apply.sh 

```

### Note: Example production-like GitFlow branching strategy with Argo & Tekton (GitOps):

<img src="poc/doc/img/gitops.png?raw=true" width="800">

<img src="poc/doc/img/GIT-2-git-glow_branching_strategy.png?raw=true" width="600">

```
### Makefile
.PHONY: build release-major release-minor release-patch

build:
	mvn ..... 

release-major:
	$(eval MAJORVERSION=$(shell git describe --tags --abbrev=0 | sed s/v// | awk -F. '{print $$1+1".0.0"}'))
	git checkout master
	git pull
	git tag -a $(MAJORVERSION) -m 'Release $(MAJORVERSION)'
	git push origin --tags

release-minor:
	$(eval MINORVERSION=$(shell git describe --tags --abbrev=0 | sed s/v// | awk -F. '{print $$1"."$$2+1".0"}'))
	git checkout master
	git pull
	git tag -a $(MINORVERSION) -m 'Release $(MINORVERSION)'
	git push origin --tags

release-patch:
	$(eval PATCHVERSION=$(shell git describe --tags --abbrev=0 | sed s/v// | awk -F. '{print $$1"."$$2"."$$3+1}'))
	git checkout master
	git pull
	git tag -a $(PATCHVERSION) -m 'Release $(PATCHVERSION)'
	git push origin --tags
```

### TODO: Use jfrog for docker registry and artefacts instead of nexus and k3d docker registry or use dockerhub registry 
Note: kubectl create secret generic dockerhub --from-file=.dockerconfigjson=$HOME/.docker/config.json --type=kubernetes.io/dockerconfigjson

Note: ssl passthrough has to be enabled for argocd grpc to work. The configuration provided for split ingress in argocd documentation doesn't work. UI login is successfull. However cli login doesn't work -> Argo CLI : `helm upgrade ingress-nginx ingress-nginx/ingress-nginx -n ingress-nginx --create-namespace --set controller.publishService.enabled=true --set controller.extraArgs.enable-ssl-passthrough=true` # or just make port forwarding (kubectl port-forward svc/argocd-server -n argocd 8080:443), then: $ argocd login localhost:8080 --username admin --password $ARGO_PASS

