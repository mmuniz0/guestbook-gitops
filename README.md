


# Building a Continuous Deployment GitOps Pipeline to Minikube

This demo is based on the WeaveWork Tutorial for EKS and ECR on AWS Cluster: [Building a Continuous Deployment GitOps Pipeline to EKS](https://github.com/weaveworks/guestbook-gitops)

This tutorial describes how to set up a basic GitOps deployment pipeline with Flux and GitHub actions. It uses the Kubernetes Guestbook application and also adapts a [GitHub Actions pipeline](https://github.com/github-developer/example-actions-flux-eks) originally developed by Jeremy Adams and John Bohannon of GitHub.
 

## What do we need for GitOps on Minikube

These are the tools that you’ll be working with in order to create a basic GitOps pipeline for application deployments or even for cluster components. If running this in production, you might also need an additional component to scan your base images for added security.


|         Component         	| Implementation                        	| Notes                                                                                                                               	|
|:-------------------------:	|---------------------------------------	|-------------------------------------------------------------------------------------------------------------------------------------	|
| Minikube                       	| [Minikube](https://minikube.sigs.k8s.io/docs/start/)             	| Local Kubernetes Cluster.                                                                                                         	|
| Git repo                  	| https://github.com/weaveworks/guestbook-gitops 	| A Git repository containing your application and cluster manifests files.                                                           	|
| Continuous Integration    	| [GitHub Actions](https://github.com/features/actions)                        	| Test and integrate the code - can be anything from CircleCI to GitHub actions.                                                      	|
| Continuous Delivery       	| [Flux version 2](https://toolkit.fluxcd.io/cmd/flux/)                                  	| Cluster <-> repo synchronization                                                                                                    	|
| Container Registry        	| [Docker Hub Registry](https://hub.docker.com/)            	| Can be any image registry or even a directory.                                                                                      	|


## Before you begin:

You will need to sign up for the following services:

* GitHub account
* Docker Hub Container Registry 
* Kubectl version 1.18 or newer
* [Kustomize](https://github.com/kubernetes-sigs/kustomize)

In this example, you will install the standard [Guestbook sample application](https://kubernetes.io/docs/tutorials/stateless-application/guestbook/) to Minkube. Then you’ll make a change to the look of the buttons and deploy it to Minikube with GitOps.

## Part 1: Fork the Guestbook app repository

You will need a GitHub account for this step.

* Fork and clone the repository: `weaveworks/guestbook-gitops`
* Keep the repo handy as you’ll be adding a deployment key to the repo that Flux requires as well as a few credentials once you have your ECR AWS container registry set up.

## Part 2: Create an Minikube cluster:

Follow the guide to install Minikube and start a local cluster: [Minikube - Install and Start](https://minikube.sigs.k8s.io/docs/start/)

## Part 2: Create an Docker Hub container registry Account and set up GitHub Actions Workflow

In this section, you will set up an Docker Hub container repository and a mini CI pipeline using GitHub Actions. The actions builds a new container on a `git push`, tags it with the git-sha and then pushes it to the Docker Hub registry; it also updates and commits the image tag change to your kustomize file.  Once the new image is in the repository, Flux notices the new image and then deploys it to the cluster. This entire flow will become more apparent in the next section after you’ve configured Flux.

1. Open an DockerHub account

Open an [Docker Hub account](https://hub.docker.com/) and create a repository. You can call the repository guestbook.

2. Specify secrets for DockerHub

Create the following GitHub secrets:

```
DOCKER_USER
DOCKER_PASSWORD
```

### Configure the GitHub actions workflow

View the workflow in your repo under .github/workflows/main.yml. Ensure you have the environment variables on lines 18-20 of main.yml set up properly.

```
18  DOCKER_USER: ${{secrets.DOCKER_USER}}
19  DOCKER_PASSWORD: ${{secrets.DOCKER_PASSWORD}}
20  CONTAINER_IMAGE: guestbook:${{ github.sha }}
```


## Part 3: Install Flux and start shipping

Flux keeps Kubernetes clusters in sync with configuration kept under source control like Git repositories, and automates updates to that configuration when there is new code to deploy. It is built using Kubernetes' API extension server, and can integrate with Prometheus and other core components of the Kubernetes ecosystem. Flux supports multi-tenancy and syncs an arbitrary number of Git repositories.

In this section, we’ll set up Flux to synchronize changes in the guestbook-gitops repository. This example is a very simple pipeline that demonstrates how to sync one application repo to a single cluster.


1. For MacOS users, you can install flux with Homebrew:

`brew install fluxcd/tap/flux`

For other installation options, see [Installing the Flux CLI](https://toolkit.fluxcd.io/get-started/#install-the-flux-cli).

2. Once installed, check that your EKS cluster satisfies the prerequisites:

`flux check --pre`

If successful, tt returns something similar to this:
```
► checking prerequisites
✔ kubectl 1.19.3 >=1.18.0
✔ Kubernetes 1.17.9-eks-a84824 >=1.16.0
✔ prerequisites checks passed
```

Flux supports synchronizing manifests in a single directory, but when you have a lot of YAML it is more efficient to use Kustomize to manage them. For the Guestbook example, all of the manifests were copied into a deploy directory and a kustomization file was added.  For this example, the kustomization file contains a `newTag` directive for the frontend images section of your deployment manifest:
```
images:
- name: frontend
  newName:guestbook
  newTag: new
```
As mentioned above, the Github Actions script updates the image tag in this file after the image is built and pushed, indicating to Flux that a new image is available in ECR.

But before we see that in action, let’s install Flux and its other controllers to your cluster.


* Set up a GitHub token:

In order to create the reconciliation repository for Flux, you'll need [a personal access token](https://help.github.com/en/github/authenticating-to-github/creating-a-personal-access-token-for-the-command-line) for your GitHub account that has permissions to create repositories. The token must have all permissions under repo checked off. Copy and keep your token handy and in a safe place.

* On the command line, export your GitHub personal access token and username:
```
export GITHUB_TOKEN=[your-github-token]
export GITHUB_USER=[your-github-username]
```
4. Create the Flux reconciliation repository. In this case you’ll call it fleet-infra, but you can call it anything you want.

In this step, a private repository is created and all of the controllers will also be installed to your EKS cluster. When bootstrapping a repository with Flux, [it’s also possible to apply only a sub-directory](https://toolkit.fluxcd.io/cmd/flux_create_kustomization/) in the repo and therefore connect to multiple clusters or locations on which to apply configuration. To keep things simple, this example sets the name of one cluster (minikube) as the apply path:
```
flux bootstrap github \
  --owner=$GITHUB_USER \
  --repository=fleet-infra \
  --branch=main \
  --path=minikube \
  --personal
```

Once it’s finished bootstrapping, you will see the following:
```
► connecting to github.com
✔ repository cloned
✚ generating manifests
✔ components manifests pushed
► installing components in flux-system namespace …..
deployment "source-controller" successfully rolled out
deployment "kustomize-controller" successfully rolled out
deployment "helm-controller" successfully rolled out
deployment "notification-controller" successfully rolled out
```

Check the cluster for the flux-system namespace with:

`kubectl get namespaces`

```
NAME              STATUS   AGE
default           Active   5h25m
flux-system       Active   5h13m
kube-node-lease   Active   5h25m
kube-public       Active   5h25m
kube-system       Active   5h25m
```

* Clone and then cd into the newly created private fleet-infra repository:

`git clone https://github.com/$GITHUB_USER/fleet-infra
cd fleet-infra`

* Connect the guestbook-gitops repo to the fleet-infra repo with:
```
flux create source git [guestbook-gitops] \
  --url=https://github.com/[github-user-id/guestbook-gitops] \
  --branch=master \
  --interval=30s \
  --export > ./[cluster-name]/[guestbook-gitops]-source.yaml
```
Where,

  * `[guestbook]` is the name of your app or service
  * `[cluster-name]` is the cluster name (in this case Minikube)
  * `[github-user-id/guestbook-gitops]` is the forked guestbook repository (in this case mmuniz0/guestbook-gitops)

Configure a Flux kustomization to apply to the `./deploy` directory from your new repo with:
```
flux create kustomization guestbook-gitops \
 --source=guestbook-gitops \
 --path="./deploy" \
 --prune=true \
 --validation=client \
 --interval=1h \
 --export > ./[cluster-name]/guestbook-gitops-sync.yaml
```
Commit all of the changes to the repository with:

`git add -A && git commit -m "add guestbook-gitops deploy" && git push`

`watch flux get kustomizations`

You should now see the latest revisions for the flux toolkit components as well as the guestbook-gitops source pulled and deployed to your cluster:
```
NAME            REVISION                                        SUSPENDED       READY   MESSAGE
flux-system     main/e1c2a084e398b9d36ce7f5067c44178b5cf9a126   False           True    Applied revision: main/e1c2a084e398b9d36ce7f5067c44178b5cf9a126
guestbook       master/35147c43026fec5a49ae31371ae8c046e4d5860e False           True    Applied revision: master/35147c43026fec5a49ae31371ae8c046e4d5860e
```

Check that all of the services of the guestbook are deploying and running in the cluster with:

`kubectl get pods -A`

Where you should see something like the following:
```
NAMESPACE     NAME                            READY   STATUS    RESTARTS   AGE
default       redis-master-545d695785-cjnks   1/1     Running   0          9m33s
default       redis-slave-84548fdbc-kbnj      1/1     Running   0          9m33s
default       redis-slave-84548fdbc-tqf52     1/1     Running   0          9m33s
flux          flux-75888db95c-9vsp6           1/1     Running   0          17m
flux          memcached-86869f57fd-42f5m      1/1     Running   0          17m
kube-system   aws-node-284tq                  1/1     Running   0          41m
kube-system   aws-node-csvs7                  1/1     Running   0          41m
kube-system   coredns-59dfd6b59f-24qzg        1/1     Running   0          48m
kube-system   coredns-59dfd6b59f-knvbj        1/1     Running   0          48m
kube-system   kube-proxy-25mks                1/1     Running   0          41m
kube-system   kube-proxy-nxf4n                1/1     Running   0          41m
```


4. Make a change to the guestbook app and deploy it with a `git push`

**Note:** If you’d like to see the Guestbook UI before you make a change, kick off a build by adding a space to the index.html file and pushing it to git.

Let’s make a simple change to the buttons on the app and push it to Git:

Open the `index.html` file and change line 15:
```
<button type="button" class="btn btn-primary btn-lg" ng-click="controller.onRedis()">Submit</button>
```
To:
```
<button type="button" class="btn btn-primary btn-lg btn-block" ng-click="controller.onRedis()">Submit</button>
```
Once you’ve made the change to the buttons do a, `git add`, `git commit` and `git push`.   Click on the Actions tab in your repo to watch the pipeline test, build and tag the image.

Now check to see that the new frontend images are deploying:

`kubectl get pods -A`
```
manuel@manus-notebook:/usr/bin$ kubectl get pods -A
NAMESPACE              NAME                                                    READY   STATUS    RESTARTS   AGE
default                frontend-6469cdfb7d-b2l2r                               1/1     Running   0          5m
default                frontend-6469cdfb7d-msrkm                               1/1     Running   0          5m
default                redis-master-f46ff57fd-grcbp                            1/1     Running   1          11m
default                redis-slave-7979cfdfb8-5vb87                            1/1     Running   1          11m
default                redis-slave-7979cfdfb8-mklzd                            1/1     Running   1          11m
flux-system            helm-controller-5b96d94c7f-6mxpj                        1/1     Running   2          11d
flux-system            kustomize-controller-5b95b78ddc-6z2dh                   1/1     Running   3          11d
flux-system            notification-controller-55f94bc746-f2hz2                1/1     Running   2          11d
flux-system            source-controller-78bfb8576-2hnbh                       1/1     Running   2          11d
kube-system            coredns-74ff55c5b-8x77n                                 1/1     Running   3          13d
kube-system            etcd-minikube                                           1/1     Running   3          13d
kube-system            kube-apiserver-minikube                                 1/1     Running   3          13d
kube-system            kube-controller-manager-minikube                        1/1     Running   3          13d
kube-system            kube-proxy-qbtrk                                        1/1     Running   3          13d
kube-system            kube-scheduler-minikube                                 1/1     Running   3          13d
kube-system            storage-provisioner                                     1/1     Running   18         13d
kubernetes-dashboard   dashboard-metrics-scraper-f6647bd8c-fn284               1/1     Running   1          11d
kubernetes-dashboard   kubernetes-dashboard-968bcb79-tps4s                     1/1     Running   1          11d

```
### Display the Guestbook application

Display the Guestbook frontend in your browser by retrieving the URL from the app running in the cluster with:

To get an external Ip in Minikube for a service you need to run in other terminal:

`minikube tunnel`

And the display the Guestbook frontend in your browser by retrieving the URL from the app running in the cluster with:

`kubectl get service frontend`

The response should be similar to this:
```
 NAME       TYPE           CLUSTER-IP     EXTERNAL-IP    PORT(S)        AGE
frontend   LoadBalancer   10.110.64.76   10.110.64.76   80:30822/TCP    11m
```

Now that you have Flux set up, you can keep making changes to the UI, and run the change through GitHub Actions to build and push new images to ECR. Flux will notice the new image and deploy your changes to the cluster, kicking your software development into overdrive.
