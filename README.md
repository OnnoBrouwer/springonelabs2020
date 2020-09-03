# SpringOne Workshop labs

## Kubernetes in Docker
### Install KinD
*kind is a tool for running local Kubernetes clusters using Docker container “nodes”.*
*kind was primarily designed for testing Kubernetes itself, but may be used for local development or CI.*
Source:  [KinD website](https://kind.sigs.k8s.io/) 

For this workshop you’ll need version 0.8.0 or higher
```bash
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.8.1/kind-$(uname -s)-amd64
chmod +x ./kind
mv ./kind /some-dir-in-your-PATH/kind
```

### Create a Kubernetes cluster
You’ll need to configure kind to expose port `80` to the host machine. That port will be used by Kourier (an ingress controller) a little later on

```bash
cat > clusterconfig.yaml <<EOF
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  extraPortMappings:
  - containerPort: 31080
    hostPort: 80
EOF
```

You can create a new cluster, with the above configuration using the below command.

```bash
kind create cluster --name springone --config clusterconfig.yaml
```

## Some additional tools
### kubectl
The Kubernetes command-line tool, kubectl, allows you to run commands against Kubernetes clusters. You can use kubectl to deploy applications, inspect and manage cluster resources, and view logs. You’ll be working with Kubernetes, so you’ll need the `kubectl` command line tool.

```bash
curl -LO "https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x ./kubectl
mv ./kubectl /some-dir-in-your-PATH/kubectl
```

### kn
The command line tool that manages Knative is called kn. The tool manages both the eventing and serving parts of Knative. You’ll build functions with Knative, so you’ll need the `kn` command line tool.

```bash
curl -Lo ./kn https://github.com/knative/client/releases/download/v0.16.0/kn-$(uname -s)-amd64
chmod +x ./kn
mv ./kn /some-dir-in-your-PATH/kn
```

### tkn
The command line tool that manages Tekton is called tkn. This tool allows you to access all Tekton resources needed in this workshop. You’ll build pipelines with Tekton, so you’ll need the `tkn` command line tool.

```bash
curl -LO https://github.com/tektoncd/cli/releases/download/v0.11.0/tkn_0.11.0_$(uname -s)_x86_64.tar.gz
tar xvzf tkn_0.11.0_$(uname -s)_x86_64.tar.gz -C /some-dir-in-your-PATH tkn
```

### k9s
To keep track of what’s happening in your cluster you can use K9s. K9s is a terminal based UI to interact with your Kubernetes clusters. The aim of this project is to make it easier to navigate, observe and manage your deployed applications in the wild. K9s continually watches Kubernetes for changes and offers subsequent commands to interact with your observed resources.

```bash
curl -LO https://github.com/derailed/k9s/releases/download/v0.21.4/k9s_$(uname -s)_x86_64.tar.gz
tar xvzf k9s_$(uname -s)_x86_64.tar.gz -C /some-dir-in-your-PATH k9s
```

### Octant
Octant is an open source developer-centric web interface for Kubernetes that lets you inspect a Kubernetes cluster and its applications. It provides a visual interface to managing Kubernetes that complements and extends existing tools like kubectl and kustomize and supports a variety of debugging features such as filtering labels and streaming container logs to be part of the Kubernetes development toolkit. Throughout this workshop, you can use Octant to see what’s happening in your cluster too.

```text
To install Octant:
* download the latest release from the Octant GitHub repository: https://github.com/vmware-tanzu/octant/releases/tag/v0.15.0
* extract the zipped or gzipped file
* start the octant executable
* open a webbrowser to http://127.0.0.1:7777
```


## Knative Serving
### Install Knative Serving to your cluster
You’re ready to install Knative serving. Knative components build on top of Kubernetes, abstracting away the complex details and enabling developers to focus on what matters. Built by codifying the best practices shared by successful real-world implementations, Knative solves the “boring but difficult” parts of deploying and managing cloud native services so you don’t have to. Knative Serving runs serverless containers on Kubernetes with ease, Knative takes care of the details of networking, autoscaling (even to zero), and revision tracking. You just have to focus on your core logic.

```bash
kubectl apply -f https://github.com/knative/serving/releases/download/v0.16.0/serving-crds.yaml
kubectl apply -f https://github.com/knative/serving/releases/download/v0.16.0/serving-core.yaml
kubectl wait deployment activator autoscaler controller webhook --for=condition=Available -n knative-serving --timeout=-1s
```

If you’re curious what just happened to your cluster, you can check the Octant user interface on `http://127.0.0.1:7777`
![IMAGE4](./images/image4.png)

### Setting up your ingress with Kourier
Kourier is an Ingress for  Knative Serving. Kourier is a lightweight alternative for the Istio ingress as its deployment consists only of an Envoy proxy and a control plane for it. You’ll need to install Kourier in the `kourier-system` namespace.

```bash
kubectl apply -f https://github.com/knative/net-kourier/releases/download/v0.16.0/kourier.yaml
kubectl wait deployment 3scale-kourier-control 3scale-kourier-gateway --for=condition=Available -n kourier-system --timeout=-1s
```

Kind is running on your local machine, so the external IP address is your local host (`127.0.0.1`). You’ll need that external IP address in other steps as well, so it’s useful to export it to a variable.

```bash
export EXTERNAL_IP="127.0.0.1"
```

nip.io is a dead simple wildcard DNS for any IP Address.  nip.io  allows you to do that by mapping any IP Address to a hostname, which means you can stop editing your etc/hosts file with custom hostname and IP address mappings. Using the external IP address, you can use nip.io to make sure any DNS query resolves to that IP address

```bash
export KNATIVE_DOMAIN="$EXTERNAL_IP.nip.io"
```

To validate it works properly, you can use the `dig` command.

```bash
dig $KNATIVE_DOMAIN
```

Now you need to update the DNS settings for Knative to make sure that Knative knows about the DNS settings. To do that, you’ll need to patch the `config-domain` config map.

```bash
kubectl patch configmap -n knative-serving config-domain -p "{\"data\": {\"$KNATIVE_DOMAIN\": \"\"}}"
```

To validate that command was executed successfully, you can `describe` the config map.

```bash
kubectl describe configmap -n knative-serving config-domain
```

Now that the configuration is almost done, you’ll need to configure Kourier to listen for http traffic on port 80 of the kind node and change the service type to NodePort instead of LoadBalancer.

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: kourier-ingress
  namespace: kourier-system
  labels:
    networking.knative.dev/ingress-provider: kourier
spec:
  type: NodePort
  selector:
    app: 3scale-kourier-gateway
  ports:
    - name: http2
      nodePort: 31080
      port: 80
      targetPort: 8080
EOF
```

The last step is to update the configuration of Knative to use Kourier

```bash
kubectl patch configmap -n knative-serving --type merge config-network -p '{"data":{"ingress.class":"kourier.ingress.networking.knative.dev"}}'
```

To validate that command was executed successfully, you can `describe` the config map

```bash
kubectl describe configmap -n knative-serving config-network
```

To make sure everything is okay, all pods should be in `Running` state and your `kourier-ingress` service configured.

```bash
kubectl get pods -n knative-serving
kubectl get pods -n kourier-system
kubectl get svc  -n kourier-system kourier-ingress
```

### Deploying your first app
You’re now ready to deploy our first application, and play around with scaling to zero and traffic splitting (super useful if you want to do A/B testing or roll out features that shouldn’t get traffic yet). To show these concepts you’ll use the hello-world sample from Knative. For now, we’ll set the autoscale window to 15 seconds, which means that the service is scaled to zero if no request was received during that time.

```bash
kn service create hello --image gcr.io/knative-samples/helloworld-go --autoscale-window 15s
```

You can check that the service was created with the below command.

```bash
kn service list hello
```

In a separate terminal window, run the below command to see how the containers are created and terminated when you invoke your app.

```bash
kubectl get pod -l serving.knative.dev/service=hello -w
```

You can call your app using the below curl command.

```bash
curl http://hello.default.127.0.0.1.nip.io
```

### Dynamic scaling
```text
Using your favorite load testing tool, try to send as much traffic as you can to that URL to see the dynamic scaling in action.
```

### Updating your app
Saying hello to the world is nice, but saying it to the rest of the SpringOne conference is even better 😉 Let’s update the environment variable `TARGET` with a new message.

```bash
kn service update hello --env TARGET="SpringOne!"

### Note: if you’re using zsh as your terminal you might need to escape the exclamation point (\!)
```

You can call your app using the same URL and see a different message

```bash
curl http://hello.default.127.0.0.1.nip.io
```

### Viewing resources
Within Octant you can see all revisions under “`custom resources -> revisions.serving.knative.dev`”. From a Knative point of view, these resources are immutable so you cannot make any changes. It does however, give you an amazing overview of all the Kubernetes objects that are created when you run the `kn service create` command.
![IMAGE7](./images/image7.png)

You can create new services using the Octant UI as well. Click on “`Apply YAML`” in the top right corner and paste the below snippet. After that, hit apply to create a new Knative service.

```yaml
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: octant-knative
  namespace: default
spec:
  template:
    spec:
      containers:
        - image: gcr.io/knative-samples/helloworld-go
          env:
            - name: TARGET
              value: "Octant"
```

You can call your app using the same URL and see a different message:

```bash
curl http://octant-knative.default.127.0.0.1.nip.io
```

### Splitting traffic
But what if you’re not entirely sure that this is the right message? That’s where A_B testing comes in. A/B testing (also known as bucket testing or split-run testing) is a user experience research methodology. A/B tests consist of a randomized experiment with two variants, A and B. To do that, we can split the traffic between the first version of the service (which said hello to the world) and the second one (which said hello to SpringOne)

First, you’ll need to `tag` the first version of the service. You can get a list of all revisions (versions) with the below command.

```bash
kn revision list
```

The first version, which will be on the bottom of the list (with no value in the traffic column) is the first version. To tag that one, run the below command.

```bash
kn service update hello --tag <revision name>=v1
```

To set up traffic splitting between the tagged V1 version and the latest version (for a `75/25` split), you can run the below command.

```bash
kn service update hello --traffic v1=75,@latest=25
```

The commands `kn revision list` and `kn service describe hello` should now show you a split in traffic. To see that in action, run the below command.

```bash
while true; do
curl http://hello.default.127.0.0.1.nip.io
sleep 0.5
done

### You can try setting different percentages to see the changes in the response to the above command.
```

This is great! Now let’s prepare for VMworld 2020, next month… The new version should be ready and testable, though you don’t want to expose this version to everyone just yet. To do that, you’ll have to create a new revision and split traffic accordingly.

You’ll need to tag the current version (V2) first

```bash
kn service update hello --tag $(kubectl get ksvc hello --template='{{.status.latestReadyRevisionName}}')=v2
```

Now, let’s create a new version for VMworld 2020, tag it with V3 (you’ll see why in just a second) and split traffic so that it doesn’t get any traffic at all.

```bash
kn service update hello --env TARGET="VMworld 2020!" --tag @latest=v3 --traffic v1=75,v2=25,@latest=0
```

The commands `kn revision list` and `kn service describe hello` should show you the same split in traffic as before (and also show the new version getting no traffic). If you want to try it out, you can use the below command.

```bash
while true; do
curl http://hello.default.127.0.0.1.nip.io
sleep 0.5
done
```

You can access specific versions of the service with `http://<tag>-hello.default.127.0.0.1.nip.io`. So even though V3 of the service isn’t getting any traffic you can send it a request with the below curl command.

```bash
curl http://v3-hello.default.127.0.0.1.nip.io
```

Now let’s assume that we’re at VMworld 2020 and need to get this version of the service live and direct all traffic here. You can do that with the below command.

```bash
kn service update hello --traffic @latest=100
```

The commands `kn revision list` and `kn service describe hello` should now show you that all traffic is sent to version 3. If you want to try it out, you can use the below command.

```bash
while true; do
curl http://hello.default.127.0.0.1.nip.io
sleep 0.5
done
```

You can still access the older versions with `curl http://v1-hello.default.127.0.0.1.nip.io` or  `curl http://v2-hello.default.127.0.0.1.nip.io`

To get a complete YAML overview of everything that you’ve done so far, you can use the experimental `export --with-revisions` command.

```bash
kn service export hello --with-revisions --mode=export -o yaml
```

## Tekton
### Install Tekton
So far you’ve done a lot manually. Let’s go and automate some of this! Tekton is a powerful yet flexible Kubernetes-native open-source framework for creating continuous integration and delivery (CI/CD) systems. It lets you build, test, and deploy across multiple cloud providers or on-premises systems by abstracting away the underlying implementation details. You’ll need to install Tekton in the namespace `tekton-pipelines`.

```bash
kubectl apply -f https://github.com/tektoncd/pipeline/releases/download/v0.14.1/release.yaml
kubectl wait deployment tekton-pipelines-controller tekton-pipelines-webhook --for=condition=Available -n tekton-pipelines
```

Some cool features of Tekton include that Tekton Pipelines are `Cloud Native`:
* Run on Kubernetes
* Have Kubernetes clusters as a first class type
* Use containers as their building blocks

Tekton Pipelines are `Decoupled`:
* One Pipeline can be used to deploy to any k8s cluster
* The Tasks which make up a Pipeline can easily be run in isolation
* Resources such as git repos can easily be swapped between runs

To get some visibility into what Tekton is doing, you can install the Tekton dashboard too

```bash
kubectl apply -f https://github.com/tektoncd/dashboard/releases/download/v0.7.1/tekton-dashboard-release.yaml
kubectl wait deployment tekton-dashboard --for=condition=Available -n tekton-pipelines
```

You can use Kourier to make the dashboard accessible to your host on the URL http://dashboard.tekton-pipelines.127.0.0.1.nip.io. If you want a different URL, you can change the “`hosts`” element in the below command.

```bash
cat <<EOF | kubectl apply -f -
apiVersion: networking.internal.knative.dev/v1alpha1
kind: Ingress
metadata:
  name: tekton-dashboard
  namespace: tekton-pipelines
  annotations:
    networking.knative.dev/ingress.class: kourier.ingress.networking.knative.dev
spec:
  rules:
  - hosts:
    - dashboard.tekton-pipelines.$KNATIVE_DOMAIN
    http:
      paths:
      - splits:
        - appendHeaders: {}
          serviceName: tekton-dashboard
          serviceNamespace: tekton-pipelines
          servicePort: 9097
    visibility: ExternalIP
  visibility: ExternalIP
EOF
```

### Setting up your container registry
To publish container images you’ll need to have access to a registry. In this workshop you’ll use Docker Hub, but you can replace that with a different registry if you want as well. You’ll use a Kubernetes secret of type `docker-registry` with the name `dockercreds`.

```bash
kubectl create secret docker-registry dockercreds --docker-server="docker.io" --docker-username="" --docker-password="" 
```

You can see the result of the command with `kubectl describe secret dockercreds`.

If you want to do this manually, you can use the below command.

```
export USERNAME=`echo -n "<your docker username>" | base64`
export PASSWORD=`echo -n "<your docker password>" | base64`
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: dockercreds
  annotations:
    tekton.dev/docker-0: https://index.docker.io/v1/
type: kubernetes.io/basic-auth
data:
  ## Use 'echo -n "username" | base64' to generate this string
  username: $USERNAME
  ## Use 'echo -n "password" | base64' to generate this string
  password: $PASSWORD

EOF
```

The next step is to create a Kubernetes ServiceAccount that will have access to the secret so it can publish images.

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: pipeline
secrets:
  - name: dockercreds
EOF
```

The command `kubectl describe serviceaccount pipelin`` shows all configured items for the service account, including that it has access to a *Mountable Secret* called dockercreds. To make sure the ServiceAccount has the right RBAC permissions to deploy services to the namespace `default`, you’ll have to set up the correct permissions.

```bash
cat <<EOF | kubectl apply -f -
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pipeline
  namespace: default
rules:
  - apiGroups:
      - serving.knative.dev
    resources:
      - "*"
    verbs:
      - create
      - update
      - patch
      - delete
      - get
      - list
      - watch
  - apiGroups:
      - ""
    resources:
      - configmaps
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - triggers.tekton.dev
    resources:
      - "*"
    verbs:
      - create
      - update
      - patch
      - delete
      - get
      - list
      - watch
  - apiGroups:
      - tekton.dev
    resources:
      - "*"
    verbs:
      - create
      - update
      - patch
      - delete
      - get
      - list
      - watch
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: pipeline
  namespace: default
subjects:
  - kind: ServiceAccount
    name: pipeline
    namespace: default
roleRef:
  kind: Role
  name: pipeline
  apiGroup: rbac.authorization.k8s.io
EOF
```

### Build a Tekton Task
You’ll incrementally build up the Tekton Task that builds the image and try it out every time to see the results. The first step is to clone the source code.

```bash
cat <<EOF | kubectl apply -f -
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: build
spec:
  params:
    - name: repo-url
      description: The git repository url
    - name: revision
      description: The branch, tag, or git reference from the git repo-url location
      default: master
  steps:
    - name: git-clone
      image: alpine/git
      script: |
        git clone \$(params.repo-url) /source
        cd /source
        git checkout \$(params.revision)
      volumeMounts:
        - name: source
          mountPath: /source
  volumes:
    - name: source
      emptyDir: {}
EOF
```

You can get a detailed overview of the data in the task with `tkn task ls` and `tkn task describe build`. You can also view the task in the Tekton dashboard.

*For the source code, you can use an existing repository https://github.com/retgits/simple-node-app, or you can make a fork of that repository to use. If you’re using your own forked version, make sure you update the URLs in the snippets accordingly.*

Let’s use the Tekton CLI to test the task. You’ll need to pass the ServiceAccount `pipeline` to be used to run the Task and you’ll need to pass in the URL to a GitHub repo you want to use.

```bash
tkn task start build --showlog \
  -p repo-url=https://github.com/retgits/simple-node-app \
  -s pipeline 
```

Next, you’ll add the build step that builds a docker image and pushes it to DockerHub

```bash
cat <<EOF | kubectl apply -f -
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: build
spec:
  params:
    - name: repo-url
      description: The git repository url
    - name: revision
      description: The branch, tag, or git reference from the git repo-url location
      default: master
    - name: image
      description: Name (reference) of the image to build.
  steps:
    - name: git-clone
      image: alpine/git
      script: |
        git clone \$(params.repo-url) /source
        cd /source
        git checkout \$(params.revision)
      volumeMounts:
        - name: source
          mountPath: /source
    - name: build-image
      image: quay.io/buildah/stable:v1.14.8
      workingdir: /source
      script: |
        echo "Building Image \$(params.image)"
        buildah --storage-driver=overlay bud --format=docker --tls-verify=false -f ./Dockerfile -t \$(params.image) .
        echo "Pushing Image \$(params.image)"
        buildah  --storage-driver=overlay push --tls-verify=false --digestfile ./image-digest \$(params.image) docker://\$(params.image)
      securityContext:
        privileged: true
      volumeMounts:
        - name: varlibcontainers
          mountPath: /var/lib/containers
        - name: source
          mountPath: /source
  volumes:
    - name: varlibcontainers
      emptyDir: {}
    - name: source
      emptyDir: {}
EOF
```

You can get a detailed overview of the data in the task with `tkn task ls` and `tkn task describe build`. You can also view the task in the Tekton dashboard.

Let’s use the Tekton CLI to test the task. You’ll need to pass the ServiceAccount `pipeline` to be used to run the Task and you’ll need to pass in the URL to a GitHub repo you want to use.

```bash
tkn task start build --showlog \
  -p repo-url=https://github.com/retgits/simple-node-app \
  -p image=docker.io/retgits/springoneapp \
  -s pipeline 
```

Now you’ve built an image, but it isn’t deployed yet. Let’s do that next

```bash
cat <<EOF | kubectl apply -f -
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: build
spec:
  params:
    - name: repo-url
      description: The git repository url
    - name: revision
      description: The branch, tag, or git reference from the git repo-url location
      default: master
    - name: image
      description: Name (reference) of the image to build.
    - name: service
      description: Name of the service to deploy
  steps:
    - name: git-clone
      image: alpine/git
      script: |
        git clone \$(params.repo-url) /source
        cd /source
        git checkout \$(params.revision)
      volumeMounts:
        - name: source
          mountPath: /source
    - name: build-image
      image: quay.io/buildah/stable:v1.14.8
      workingdir: /source
      script: |
        echo "Building Image \$(params.image)"
        buildah --storage-driver=overlay bud --format=docker --tls-verify=false -f ./Dockerfile -t \$(params.image) .
        echo "Pushing Image \$(params.image)"
        buildah  --storage-driver=overlay push --tls-verify=false --digestfile ./image-digest \$(params.image) docker://\$(params.image)
      securityContext:
        privileged: true
      volumeMounts:
        - name: varlibcontainers
          mountPath: /var/lib/containers
        - name: source
          mountPath: /source
    - name: kn-deploy
      image: gcr.io/knative-releases/knative.dev/client/cmd/kn:latest
      args:
      - service
      - create
      - --force
      - \$(params.service)
      - --image=\$(params.image)
  volumes:
    - name: varlibcontainers
      emptyDir: {}
    - name: source
      emptyDir: {}
EOF
```

You can get a detailed overview of the data in the task with `tkn task ls` and `tkn task describe build`. You can also view the task in the Tekton dashboard.

Let’s use the Tekton CLI to test the task. In addition to the parameters from the last run, you’ll also need to pass in the name of the service to create.

```bash
tkn task start build --showlog \
  -p repo-url=https://github.com/retgits/simple-node-app \
  -p image=docker.io/retgits/springoneapp \
  -p service=springoneapp \
  -s pipeline 
```

When you’re running these commands, you can also view the logs in the Tekton dashboard on `http://dashboard.tekton-pipelines.127.0.0.1.nip.io`

This is great! Your service is now deployed on Knative and ready to accept traffic. You can view the service with `kn service list` and you can access the app with curl `curl http://springoneapp.default.127.0.0.1.nip.io`

```text
In exploring Knative services, you used an environment variable to change the message. Can you add that variable in the Tekton task too?
```

```text
When you have multiple versions (revisions) of the service, how can you extend this pipeline to split traffic?
```

### Setting up pipelines
The current task is quite large. It has 4 steps that essentially do two things, build an image and deploy it as a Knative service. In this section you’ll break up the Task into two smaller tasks, create a pipeline to combine them, and run the pipeline. As you’re running these commands, know that you can also view the logs in the Tekton dashboard on `http://dashboard.tekton-pipelines.127.0.0.1.nip.io`

```bash
cat <<EOF | kubectl apply -f -
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: git-build
spec:
  params:
    - name: repo-url
      description: The git repository url
    - name: revision
      description: The branch, tag, or git reference from the git repo-url location
      default: master
    - name: image
      description: Name (reference) of the image to build.
  steps:
    - name: git-clone
      image: alpine/git
      script: |
        git clone \$(params.repo-url) /source
        cd /source
        git checkout \$(params.revision)
      volumeMounts:
        - name: source
          mountPath: /source
    - name: build-image
      image: quay.io/buildah/stable:v1.14.8
      workingdir: /source
      script: |
        echo "Building Image \$(params.image)"
        buildah --storage-driver=overlay bud --format=docker --tls-verify=false -f ./Dockerfile -t \$(params.image) .
        echo "Pushing Image \$(params.image)"
        buildah  --storage-driver=overlay push --tls-verify=false --digestfile ./image-digest \$(params.image) docker://\$(params.image)
      securityContext:
        privileged: true
      volumeMounts:
        - name: varlibcontainers
          mountPath: /var/lib/containers
        - name: source
          mountPath: /source
  volumes:
    - name: varlibcontainers
      emptyDir: {}
    - name: source
      emptyDir: {}
EOF
```

You can get a detailed overview of the data in the task with `tkn task ls` and `tkn task describe git-build`. You can also view the task in the Tekton dashboard.

```bash
cat <<EOF | kubectl apply -f -
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: ksvc-deploy
spec:
  params:
    - name: image
      description: Name (reference) of the image to build.
    - name: service
      description: Name of the service to deploy
  results:
    - name: url
      description: URL of the Knative service deployed
  steps:
    - name: kn-deploy
      image: gcr.io/knative-releases/knative.dev/client/cmd/kn:latest
      args:
      - service
      - create
      - --force
      - \$(params.service)
      - --image=\$(params.image)
    - name: kn-url
      image: gcr.io/knative-releases/knative.dev/client/cmd/kn:latest
      args:
      - service
      - describe
      - \$(params.service)
      - --output=jsonpath={.status.url} > /tekton/results/url
EOF
```

You can get a detailed overview of the data in the task with `tkn task ls` and `tkn task describe ksvc-deploy`. You can also view the task in the Tekton dashboard. The `ksvc-deploy` task can also be used to deploy any other container as a Knative service.

```bash
tkn task start ksvc-deploy --showlog \
  -p image=gcr.io/knative-samples/helloworld-go \
  -p service=helloworld \
  -s pipeline 
```

Now for the Tekton pipeline

```bash
cat <<EOF | kubectl apply -f -
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: build-and-deploy
spec:
  params:
    - name: repo-url
      description: The git repository url
    - name: revision
      description: The branch, tag, or git reference from the git repo-url location
      default: master
    - name: image
      description: Name (reference) of the image to build.
    - name: service
      description: Name of the service to deploy
  results:
    - name: url
      description: URL of the Knative service deployed
  tasks:
    - name: git-build
      taskRef:
        name: git-build
      params:
        - name: image
          value: \$(params.image)
        - name: repo-url
          value: \$(params.repo-url)
        - name: revision
          value: \$(params.revision)
    - name: ksvc-deploy
      runAfter: [git-build]
      taskRef:
        name: ksvc-deploy
      params:
        - name: image
          value: \$(params.image)
        - name: service
          value: \$(params.service)
EOF
```

This time you can view the result with `tkn pipeline ls` and `tkn pipeline describe build-and-deploy`. As always, the results are also available in the Tekton dashboard.

Now you get to test it out, using the Tekton CLI you can run this pipeline just like the tasks earlier.

```bash
tkn pipeline start build-and-deploy --showlog \
  -p repo-url=https://github.com/retgits/simple-node-app \
  -p image=docker.io/retgits/springoneapp \
  -p service=springoneapp \
  -s pipeline 
```

You’ll see that the output now has the name of the Pipeline step and the Task step. Within the Tekton dashboard, you can see the pipeline in the `PipelineRuns` section. The individual tasks are still viewable in the `TaskRuns` section.

In the terminal, you can get a full overview of the pipeline run as well with the below command.

```bash
tkn pipelinerun describe --last
```

### Add Tekton triggers
To make this a more evented flow, you can use the web hook trigger. You’ll trigger the web hook manually at the end since everything is running on your own machine.

The first step is to install the Tekton Triggers

```bash
kubectl apply -f https://github.com/tektoncd/triggers/releases/download/v0.6.1/release.yaml
kubectl wait deployment tekton-triggers-controller tekton-triggers-webhook --for=condition=Available -n tekton-pipelines
```

When a message for your web hook arrives, it should start a pipeline. You’ll need to create a trigger template to instruct Tekton to create a PipelineRun. Make sure that you update the default image name before applying the YAML.

```bash
cat <<EOF | kubectl apply -f -
apiVersion: triggers.tekton.dev/v1alpha1
kind: TriggerTemplate
metadata:
  name: build-and-deploy
spec:
  params:
    - name: revision
      description: The git revision.
      default: master
    - name: repo-url
      description: The git repository url.
    - name: repo-name
      description: The git repository name.
    - name: image
      description: Name (reference) of the image to build.
      default: docker.io/retgits/springoneapp
  resourcetemplates:
    - apiVersion: tekton.dev/v1beta1
      kind: PipelineRun
      metadata:
        generateName: build-and-deploy-run
      spec:
        serviceAccountName: pipeline
        pipelineRef:
          name: build-and-deploy
        params:
          - name: revision
            value: \$(params.revision)
          - name: repo-url
            value: \$(params.repo-url)
          - name: image
            value: \$(params.image)
          - name: service
            value: \$(params.repo-name)
EOF
```

When the Webhook invokes we want to extract information from the Web Hook http request sent by the Git Server, we will use a TriggerBinding this information is what gets passed to the TriggerTemplate.

```bash
cat <<EOF | kubectl apply -f -
apiVersion: triggers.tekton.dev/v1alpha1
kind: TriggerBinding
metadata:
  name: build-and-deploy
spec:
  params:
    - name: revision
      value: \$(body.head_commit.id)
    - name: repo-url
      value: \$(body.repository.url)
    - name: repo-name
      value: \$(body.repository.name)
EOF
```

To be able to handle the http request sent by the GitHub Webhook, we need a webserver. Tekton provides a way to define this listeners that takes the TriggerBinding and the TriggerTemplate as specification. We can specify Interceptors to handle any customization for example I only want to start a new `Pipeline` only when push happens on the main branch

```bash
cat <<EOF | kubectl apply -f -
apiVersion: triggers.tekton.dev/v1alpha1
kind: EventListener
metadata:
  name: cicd
spec:
  serviceAccountName: pipeline
  triggers:
    - name: cicd-trigger
      bindings:
        - ref: build-and-deploy
      template:
        name: build-and-deploy
      interceptors:
        - cel:
            filter: "header.match('X-GitHub-Event', 'push') && body.ref == 'refs/heads/master'"
EOF
```

The Eventlister creates a deployment and a service you can list both using this command `kubectl get deployments,eventlistener,svc -l eventlistener=cicd`. The deployment should show `READY 1/1` before you can continue with the rest of this exercise.

*Note: For this workshop, you’ll trigger the web hook manually, but if you’re running on a system that can receive traffic from the Internet or if you’re using a tunneling tool like Inlets or Ngrok you can set up an actual web hook to be triggered from GitHub*

The first step is to expose the event listener you just deployed through Kourier (very similar to what you’ve done with the Tekton dashboard)

```bash
cat <<EOF | kubectl apply -f -
apiVersion: networking.internal.knative.dev/v1alpha1
kind: Ingress
metadata:
  name: el-cicd
  namespace: default
  annotations:
    networking.knative.dev/ingress.class: kourier.ingress.networking.knative.dev
spec:
  rules:
  - hosts:
    -  el-cicd.default.$KNATIVE_DOMAIN
    http:
      paths:
      - splits:
        - appendHeaders: {}
          serviceName: el-cicd
          serviceNamespace: default
          servicePort: 8080
    visibility: ExternalIP
  visibility: ExternalIP
EOF
```

You can export the web hook as a terminal variable

```bash
export GIT_WEBHOOK_URL=http://el-cicd.default.$KNATIVE_DOMAIN
```

In the spirit of security, please be aware that the URL is HTTP and NOT https. It’s not a secure URL so please do not use this for cases where you want security.

```bash
curl -v --request POST \
  --url http://el-cicd.default.127.0.0.1.nip.io/ \
  --header 'content-type: application/json' \
  --header 'x-github-event: push' \
  --data '{
    "ref": "refs/heads/master",
    "repository": {
        "url": "https://github.com/retgits/simple-node-app",
				"name": "simple-node-app"
    },
    "head_commit": {
        "id": "7919de41971aa3abc6c218e7a823da95861c5a3e"
    }
}'
```

A new Tekton `PipelineRun` gets created starting a new `Pipeline` Instance. You can check in the Tekton Dashboard for progress of use the tkn CLI

```bash
tkn pipeline logs -f --last
```

And to see details of the run, use

```bash
tkn pipelinerun describe --last
```

## Knative Eventing 
### Install Knative Eventing to your cluster
So far everything you’ve done is with HTTP, in the next section you’ll look at the Eventing side of Knative. To start you’ll need to install Knative Eventing

```bash
kubectl apply --filename https://github.com/knative/eventing/releases/download/v0.16.0/eventing-crds.yaml
kubectl apply --filename https://github.com/knative/eventing/releases/download/v0.16.0/eventing-core.yaml
kubectl wait deployment eventing-controller eventing-webhook --for=condition=Available -n knative-eventing --timeout=-1s
```

Next you’ll need to install a messaging layer, or Channel in Knative Eventing terms, and for this workshop you’ll use the In-Memory channel. This implementation is nice because it is simple and standalone, but it is unsuitable for production use cases.

```bash
kubectl apply --filename https://github.com/knative/eventing/releases/download/v0.16.0/in-memory-channel.yaml
kubectl wait deployment imc-controller imc-dispatcher --for=condition=Available -n knative-eventing --timeout=-1s
```

The final step to get things running is to install a broker. This eventing layer utilizes Channels and runs event routing components in a System Namespace. If you’ve used Knative Eventing in the past, you’ll notice that this is a little different from previous versions, because the broker now provides a smaller and simpler installation.

```bash
kubectl apply --filename https://github.com/knative/eventing/releases/download/v0.16.0/mt-channel-broker.yaml
kubectl wait deployment mt-broker-controller mt-broker-filter mt-broker-ingress --for=condition=Available -n knative-eventing --timeout=-1s
```

To make sure everything is okay, all pods should be in Running state.

```bash
kubectl get pods --namespace knative-eventing
```

### Deploying your first event-driven app
To use Knative Eventing in a namespace, you’ll need to make sure that there’s a broker running. The below command will create a new broker called `default` in the namespace `default`.

```bash
kubectl label namespace default knative-eventing-injection=enabled

cat <<EOF | kubectl apply -f -
apiVersion: eventing.knative.dev/v1
kind: Broker
metadata:
  name: default
  namespace: default
EOF
```

To check that everything has worked, you can run the command `kubectl get broker`

To visualize the events, you’ll need to have an event display. While Knative has a default (and tools like  [Sockeye](https://github.com/n3wscott/sockeye)  exist as well), let’s create one from scratch.

```python
cat > monitor.py << EOF
import json
import logging
import os

from flask import Flask, request

app = Flask(__name__)

@app.route('/', methods=['POST'])
def pubsub_push():
    message = json.loads(request.data.decode('utf-8'))
    info(f'Event Display received message:\n{message}')
    return 'OK', 200

def info(msg):
    app.logger.info(msg)

if __name__ != '__main__':
    ## Redirect Flask logs to Gunicorn logs
    gunicorn_logger = logging.getLogger('gunicorn.error')
    app.logger.handlers = gunicorn_logger.handlers
    app.logger.setLevel(gunicorn_logger.level)
else:
    app.run(debug=True, host='0.0.0.0', port=int(os.environ.get('PORT', 8080)))
EOF
```

You’ll need to create a docker container out of that

```bash
cat > Dockerfile << EOF
FROM python:3.7-slim

RUN pip install Flask gunicorn

WORKDIR /app
COPY monitor.py .

CMD exec gunicorn --bind :\$PORT --workers 1 --threads 8 monitor:app
EOF

docker build . -t <docker username>/display
docker push <docker username>/display
```

If you’re curious what that looks like, you can run a container and see:

```bash
docker run --rm -it -e PORT=8080 -p 8080:8080 <docker username>/display:latest
```

And in a different terminal window, run

```bash
curl --request POST \
  --url http://localhost:8080/ \
  --header 'content-type: application/json' \
  --data '{
	"hello": "springone"
}'
```

We’re now ready to deploy the display. You’ll need to keep it running, so the minimal number of replicas needs to be set to `1`

```bash
kn service create event-display --image <docker username>/display:latest --min-scale 1
```

You can check that the service was created with the below command.

```bash
kn service list event-display
```

You’ll need to connect the display to the Broker with a trigger. The trigger makes sure that the event received on the broker is sent to the right service (in Knative terminology, these are called event sinks). In the trigger is a filter attribute of `type: springoneworkshop`. That means only events with that particular field set will be sent to the display. The subscriber  is a reference to the event-display service you deployed just now.

```bash
cat <<EOF | kubectl apply -f -
apiVersion: eventing.knative.dev/v1beta1
kind: Trigger
metadata:
  name: trigger-display
spec:
  filter:
    attributes:
      type: springoneworkshop
  subscriber:
    ref:
      apiVersion: serving.knative.dev/v1
      kind: Service
      name: event-display
EOF
```

You can validate the trigger has been created successfully with `kubectl get trigger`

To get some events flowing, you’d normally open up your ingress routes. In this case, you’ll create a pod that will act as an event producer and send events using the curl command.

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: curl
  name: curl
spec:
  containers:
  - image: radial/busyboxplus:curl
    imagePullPolicy: IfNotPresent
    name: curl
    resources: {}
    stdin: true
    terminationMessagePath: /dev/termination-log
    terminationMessagePolicy: File
    tty: true
EOF
```

In a different terminal window, you’ll tail the logs to see what’s happening. To start you’ll need the name of the pod of the display service

```bash
$ kubectl get pods -n default
NAME                                                READY   STATUS    RESTARTS   AGE
curl                                                1/1     Running   0          80s
event-display-dphdw-1-deployment-6c6695864f-ds9g2   2/2     Running   0          8m50s
```

To stream the logs, run the below command

```bash
kubectl logs -f event-display-dphdw-1-deployment-6c6695864f-ds9g2 -c user-container
```

In your first terminal window, now create an SSH session to the curl pod

```bash
kubectl attach curl -it
```

From there you can send an event using curl. This will give you a message in the logs

```bash
curl -v "http://broker-ingress.knative-eventing.svc.cluster.local/default/default" \
  -X POST \
  -H "Ce-Id: workshop" \
  -H "Ce-Specversion: 1.0" \
  -H "Ce-Type: springoneworkshop" \
  -H "Ce-Source: curl-pod" \
  -H "Content-Type: application/json" \
  -d '{"msg":"Hello SpringOne Workshop"}'
```

Similarly, the below message will not give you a log entry because the type doesn’t match

```bash
curl -v "http://broker-ingress.knative-eventing.svc.cluster.local/default/default" \
  -X POST \
  -H "Ce-Id: workshop" \
  -H "Ce-Specversion: 1.0" \
  -H "Ce-Type: helloworld" \
  -H "Ce-Source: curl-pod" \
  -H "Content-Type: application/json" \
  -d '{"msg":"Hello SpringOne Workshop"}'
```

### Events without Brokers and Triggers
As a best practice, you want to leverage brokers and triggers for all your eventing. It makes it a lot more straight-forward when you need to update implementations or add some additional subscribers. However, it does work without brokers and triggers too…

To simulate some events, you’ll use the `PingSource`. That event source is capable of sending CloudEvents events based on a cron schedule. You can use your existing display service to receive and log the events

```bash
cat <<EOF | kubectl apply -f -
apiVersion: sources.knative.dev/v1alpha2
kind: PingSource
metadata:
  name: source
spec:
  schedule: "* * * * *"
  jsonData: '{"message": "Hello SpringOne World!"}'
  sink:
    ref:
      apiVersion: serving.knative.dev/v1
      kind: Service
      name: event-display
EOF
```

This schedule will send a message every minute, so for now you’ll have to wait a little to see the output…

The reason you usually want to leverage brokers and triggers is that in case you now want to add a second display function to receive the messages, you’d need to update the PingSource deployment. Instead, with triggers and brokers you could add a new trigger.

### More than one source
Let’s update the previous example a little to make sure it follows best practices (and you’ll add a second receiver of the message too). The first step is to remove the PingSource, because you’ll create a new one later on

```bash
kubectl delete pingsource.sources.knative.dev source
```

The first step is to create a Channel. In Knative terminology, a channel takes care of message persistence and forwarding. In this case, you’ll use the InMemory channel which isn’t persistent but as you’re not running $10M transactions over your machine right now that’s probably okay.

```bash
cat <<EOF | kubectl apply -f -
apiVersion: messaging.knative.dev/v1beta1
kind: InMemoryChannel
metadata:
  name: channel
EOF
```

The new PingSource will leverage the channel to send events to

```bash
cat <<EOF | kubectl apply -f -
apiVersion: sources.knative.dev/v1alpha2
kind: PingSource
metadata:
  name: source
spec:
  schedule: "* * * * *"
  jsonData: '{"message": "Hello SpringOne World Channel!"}'
  sink:
    ref:
      apiVersion: messaging.knative.dev/v1beta1
      kind: InMemoryChannel
      name: channel
EOF
```

Let’s add a second display. You’ll need to keep it running as well, so the minimal number of replicas needs to be set to `1`

```bash
kn service create event-display2 --image <docker username>/display:latest --min-scale 1
```

The command `kn service list` should now show two services running

The next step is to create subscriptions. These subscriptions tie the services to the channel and let the channel know which services need to get messages

```bash
cat <<EOF | kubectl apply -f -
apiVersion: messaging.knative.dev/v1beta1
kind: Subscription
metadata:
  name: sub1
spec:
  channel:
    apiVersion: messaging.knative.dev/v1beta1
    kind: InMemoryChannel
    name: channel
  subscriber:
    ref:
      apiVersion: serving.knative.dev/v1
      kind: Service
      name: event-display
EOF

cat <<EOF | kubectl apply -f -
apiVersion: messaging.knative.dev/v1beta1
kind: Subscription
metadata:
  name: sub2
spec:
  channel:
    apiVersion: messaging.knative.dev/v1beta1
    kind: InMemoryChannel
    name: channel
  subscriber:
    ref:
      apiVersion: serving.knative.dev/v1
      kind: Service
      name: event-display2
EOF
```

As soon as you create the subscriptions, the messages are received by the event display services. You can verify that in the same way as before by running the logs command. To start you’ll need the name of the pod of the display services

```bash
$ kubectl get pods -n default
NAME                                                 READY   STATUS    RESTARTS   AGE
curl                                                 1/1     Running   2          33m
event-display-dphdw-1-deployment-6c6695864f-ds9g2    2/2     Running   0          41m
event-display2-lmbdd-1-deployment-77dd94f5d6-cpnc4   2/2     Running   0          5m6s
```

To stream the logs, run these commands in separate terminal windows

```bash
kubectl logs -f event-display-dphdw-1-deployment-6c6695864f-ds9g2 -c user-container

kubectl logs -f event-display2-lmbdd-1-deployment-77dd94f5d6-cpnc4 -c user-container
```

### Sequence of events
When you’re building functionality, it can be useful to split up functionality into smaller chunks. For example, in Unix operating systems the idea is that each command like tool does only one thing. Using the concepts of pipe and tee, you can string these commands together. Looking at the Pull Request example, you could have a function that strips out sensitive (Personally Identifiable Information or PII). That functionality can be very useful to other flows too, so you can create a function that does only that one thing. The output of that function feeds into other functions, chaining functions together into a longer sequence.
![IMAGE5](./images/image5png)

First, you’ll have to create the display that will show the final event (consumer 2). In this case you’ll use a different display than you’ve used so far.

```bash
cat <<EOF | kubectl create -f -
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: event-display-chain
spec:
  template:
    spec:
      containers:
        - image: gcr.io/knative-releases/knative.dev/eventing-contrib/cmd/event_display
EOF
```

Next, you’ll have to create a function that reacts to the event (consumer 1) and sends it onwards while modifying it a little bit too.

```bash
cat <<EOF | kubectl create -f -
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: first
spec:
  template:
    spec:
      containers:
        - image: gcr.io/knative-releases/knative.dev/eventing-contrib/cmd/appender
          env:
            - name: MESSAGE
              value: " - Updating this message a little..."
EOF
```

The sequence construct uses a channel to make sure the different steps are executed. In this case it makes sure that the service “first” is executed and that the result is passed on to the service “event-display-chain”

```bash
cat <<EOF | kubectl create -f -
apiVersion: flows.knative.dev/v1beta1
kind: Sequence
metadata:
  name: sequence
spec:
  channelTemplate:
    apiVersion: messaging.knative.dev/v1beta1
    kind: InMemoryChannel
  steps:
    - ref:
        apiVersion: serving.knative.dev/v1
        kind: Service
        name: first
  reply:
    ref:
      kind: Service
      apiVersion: serving.knative.dev/v1
      name: event-display-chain
EOF
```

Finally, you’ll need to create a PingSource (producer) that sends a new message to the sequence every minute.

```bash
cat <<EOF | kubectl create -f -
apiVersion: sources.knative.dev/v1alpha2
kind: PingSource
metadata:
  name: ping-source-sequence
spec:
  schedule: "* * * * *"
  jsonData: '{"message": "Hello Knative!"}'
  sink:
    ref:
      apiVersion: flows.knative.dev/v1beta1
      kind: Sequence
      name: sequence
EOF
```

You can check that the sequence was executed and the messages was updated by running the command below.

```bash
kubectl -n default logs -l serving.knative.dev/service=event-display-chain -c user-container --tail=-1
```

### Using the event registry
In Knative Eventing, the Event Registry maintains a catalog of the event types that can be consumed from the different Brokers. It helps you discover the different types of events you can consume from the Brokers’ event meshes.

To make sure there are events flowing through the broker, you’ll need to create a new PingSource that sends messages to the broker

```bash
cat <<EOF | kubectl apply -f -
apiVersion: sources.knative.dev/v1alpha2
kind: PingSource
metadata:
  name: brokersource
spec:
  schedule: "* * * * *"
  jsonData: '{"message": "Hello SpringOne World Broker!"}'
  sink:
    ref:
      apiVersion: eventing.knative.dev/v1
      kind: Broker
      name: default
EOF
```

The command `kubectl get eventtypes -n default` will now show the new PingSource event.  In more detail, you can use `kubectl get eventtype <name of the event> -o yaml` to see the meta data of the event. The `spec.type` field can be used to create triggers. For the final piece of this workshop, you’ll create a trigger to get messages from the broker into the display service

```bash
cat <<EOF | kubectl apply -f -
apiVersion: eventing.knative.dev/v1beta1
kind: Trigger
metadata:
  name: trigger-broker
spec:
  filter:
    attributes:
      type: dev.knative.sources.ping
  subscriber:
    ref:
      apiVersion: serving.knative.dev/v1
      kind: Service
      name: event-display
EOF
```

### Building complex eventing sequences
With the knowledge from the above sections, can you build the below diagram? It has two event sources that both can trigger function 1 and based on the original source emits an updated version of the event to a different function. Your sources should be able to be triggered by Tekton triggers. You might need to write some code to do this, and you definitely want to check everything in Octant  :)
![IMAGE6](./images/image6.png)

## Cleaning up
Now that you’re done, you can keep the cluster running or shut it down with `kind delete cluster --name springone`
