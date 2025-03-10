<p align="center">
  <a href="#google-cloud">
    <img style="width:128px;height:128px;" src="/assets/images/gcp/Google Cloud Platform.svg"/>
  </a>
  <a href="#compute-engine">
    <img style="width:128px;height:128px;" src="/assets/images/gcp/Compute Engine.svg"/>
  </a>
  <a href="#kubernetes-engine">
    <img style="width:128px;height:128px;" src="/assets/images/gcp/Container Engine.svg"/>
  </a>
  <a href="#kubernetes-engine">
    <img style="width:128px;height:128px;" src="/assets/images/gcp/Kubernetes_logo.svg" />
  </a>
</p>

<p align="center">
  <a href="#aws">
    <img style="width:128px;height:128px;" src="/assets/images/aws/AWS.svg"/>
  </a>
  <a href="#ec2">
    <img style="width:128px;height:128px;" src="/assets/images/aws/EC2.svg"/>
  </a>
  <a href="#eks">
    <img style="width:128px;height:128px;" src="/assets/images/aws/EKS.svg"/>
  </a>
</p>

<p align="center">
  <a href="#azure">
    <img style="width:128px;height:128px;" src="/assets/images/azure/Microsoft Azure.svg"/>
  </a>
  <a href="#azure-container-instances">
    <img style="width:128px;height:128px;" src="/assets/images/azure/Azure Container Service.svg"/>
  </a>
</p>

<p align="center">
  <a href="#anka">
    <img style="height:100px;" src="/assets/images/veertu/anka-logo.png"/>
  </a>
</p>

For every [task](writing-tasks.md) Cirrus CI starts a new Virtual Machine or a new Docker Container on a given compute service.
Using a new VM or a new Docker Container each time for running tasks has many benefits:

* **Atomic changes to an environment where tasks are executed.** Everything about a task is configured in `.cirrus.yml` file including
  VM image version and Docker Container image version. After commiting changes to `.cirrus.yml` not only new tasks will use the new environment
  but also outdated branches will continue using the old configuration.
* **Reproducibility.** Fresh environment guarantees no corrupted artifacts or caches are presented from the previous tasks.
* **Cost efficiency.** Most compute services are offering per-second pricing which makes them ideal for using with Cirrus CI. 
  Also each task for repository can define ideal amount of CPUs and Memory specific for a nature of the task. No need to manage
  pools of similar VMs or try to fit workloads within limits of a given Continuous Integration systems.

To be fair there are of course some disadvantages of starting a new VM or a container for every task:

* **Virtual Machine Startup Speed.** Starting a VM can take from a few dozen seconds to a minute or two depending on a cloud provider and
  a particular VM image. Starting a container on the other hand just takes a few hundred milliseconds! But even a minute
  on average for starting up VMs is not a big inconvenience in favor of more stable, reliable and more reproducible CI.
* **Cold local caches for every task execution.** Many tools tend to store some caches like downloaded dependencies locally
  to avoid downloading them again in future. Since Cirrus CI always uses fresh VMs and containers such local caches will always
  be empty. Performance implication of empty local caches can be avoided by using Cirrus CI features like 
  [built-in caching mechanism](writing-tasks.md#cache-instruction). Some tools like [Gradle](https://gradle.org/) can 
  even take advantages of [built-in HTTP cache](writing-tasks.md#http-cache)!

Please check the list of currently supported cloud compute services below and please see what's [coming next](#coming-soon).

## Google Cloud

<p align="center">
  <a href="#google-cloud">
    <img style="width:128px;height:128px;" src="/assets/images/gcp/Google Cloud Platform.svg"/>
  </a>
  </a>
</p>

Cirrus CI can schedule tasks on several Google Cloud Compute services. In order to interact with Google Cloud APIs 
Cirrus CI needs permissions. Creating a [service account](https://cloud.google.com/compute/docs/access/service-accounts) 
is a common way to safely give granular access to parts of Google Cloud Projects. 

!!! warning "Isolation"
    We do recommend to create a separate Google Cloud project for running CI builds to make sure tests are
    isolated from production data. Having a separate project also will show how much money is spent on CI and how
    efficient Cirrus CI is :wink:

Once you have a Google Cloud project for Cirrus CI please create a service account by running the following command: 

```bash
gcloud iam service-accounts create cirrus-ci \
    --project $PROJECT_ID
```

Depending on a compute service Cirrus CI will need different [roles](https://cloud.google.com/iam/docs/understanding-roles) 
assigned to the service account. But Cirrus CI will always need permissions to act as a service account:

```bash
gcloud projects add-iam-policy-binding $PROJECT_ID \
    --member serviceAccount:cirrus-ci@$PROJECT_ID.iam.gserviceaccount.com \
    --role roles/iam.serviceAccountUser
```

Cirrus CI uses Google Cloud Storage to store logs and caches. In order to give Google Cloud Storage permissions
to the service account please run:

```bash
gcloud projects add-iam-policy-binding $PROJECT_ID \
    --member serviceAccount:cirrus-ci@$PROJECT_ID.iam.gserviceaccount.com \
    --role roles/storage.admin
```

!!! info "Default Logs Retentions Period"
    By default Cirrus CI will store logs and caches for 90 days but it can be changed by manually configuring a
    [lifecycle rule](https://cloud.google.com/storage/docs/lifecycle) for a Google Cloud Storage bucket that Cirrus CI is using.

Now we have a service account that Cirrus CI can use! It's time to let Cirrus CI know about that fact by securely providing a
private key for the service account. A private key can be created by running the following command:

```bash
gcloud iam service-accounts keys create service-account-credentials.json \
  --iam-account cirrus-ci@$PROJECT_ID.iam.gserviceaccount.com
```

At last create an [encrypted variable](writing-tasks.md#encrypted-variables) from contents of
`service-account-credentials.json` file and add it to the top of `.cirrus.yml` file:

```yaml
gcp_credentials: ENCRYPTED[qwerty239abc]
```

Now Cirrus CI can store logs and caches in Google Cloud Storage for tasks scheduled on either GCE or GKE. Please check following sections 
with additional instructions about [Compute Engine](#compute-engine) or [Kubernetes Engine](#kubernetes-engine).

!!! info "Supported Regions"
    Cirrus CI currently supports following GCP regions: `us-central1`, `us-east1`, `us-east4`, `us-west1`, `us-west2`,
    `europe-west1`, `europe-west2`, `europe-west3` and `europe-west4`.
    
    Please [contact support](../support.md) if you are intersted in support for other regions.

### Compute Engine

<p align="center">
  <a href="#compute-engine">
    <img style="width:128px;height:128px;" src="/assets/images/gcp/Compute Engine.svg"/>
  </a>
</p>

In order to schedule tasks on Google Compute Engine a service account that Cirrus CI operates via should have a necessary
role assigned. It can be done by running a `gcloud` command:

```bash
gcloud projects add-iam-policy-binding $PROJECT_ID \
    --member serviceAccount:cirrus-ci@$PROJECT_ID.iam.gserviceaccount.com \
    --role roles/compute.admin
```

Now tasks can be scheduled on Compute Engine within `$PROJECT_ID` project by configuring `gce_instance` something 
like this:

```yaml
gce_instance:
  image_project: ubuntu-os-cloud
  image_name: ubuntu-1904-disco-v20190417
  zone: us-central1-a
  cpu: 8
  memory: 40GB
  disk: 60
  use_ssd: true # default to false
  
task:
  script: ./run-ci.sh
```

!!! tip "Specify Machine Type"
    It is possible to specify a [predefined machine type](https://cloud.google.com/compute/docs/machine-types) via `type`
    field:
    
    ```yaml
    gce_instance:
      image_project: ubuntu-os-cloud
      image_name: ubuntu-1604-xenial-v20171121a
      zone: us-central1-a
      type: n1-standard-8
      disk: 20
    ```

!!! tip "Specify Image Family"
    It's also possible to specify image family instead of the concrete image name. Simply specify `image_family` field
    instead of `image_name`:

    ```yaml
    gce_instance:
      image_project: ubuntu-os-cloud
      image_family: ubuntu-1904
    ```

#### Custom VM images

Building an immutable VM image with all necessary software pre-configured is a known best practice with many benefits.
It makes sure environment where a task is executed is always the same and that no time is spent on useless work like
installing a package over and over again for every single task.

There are many ways how one can create a custom image for Google Compute Engine. Please refer to the [official documentation](https://cloud.google.com/compute/docs/images/create-delete-deprecate-private-images).
At Cirrus Labs we are using [Packer](https://www.packer.io/docs/builders/googlecompute.html) to automate building such
images. An example of how we use it can be found in [our public GitHub repository](https://github.com/cirruslabs/cirrus-images).

#### Windows Support

Google Compute Engine support Windows images and Cirrus CI can take full advantages of it by just explicitly specifying
platform of an image like this:

```yaml
gce_instance:
  image_project: windows-cloud
  image_name: windows-server-2016-dc-core-v20170913
  platform: windows
  zone: us-central1-a
  cpu: 8
  memory: 40GB
  disk: 20
  
task:
  script: run-ci.bat
```

#### FreeBSD Support

Google Compute Engine support FreeBSD images and Cirrus CI can take full advantages of it by just explicitly specifying
platform of an image like this:

```yaml
gce_instance:
  image_project: freebsd-org-cloud-dev
  image_family: freebsd-12-1
  platform: FreeBSD
  zone: us-central1-a
  cpu: 8
  memory: 40GB
  disk: 50
  
task:
  script: printenv
```

#### Docker Containers on Dedicated VMs

It is possible to run a container directly on a Compute Engine VM with pre-installed Docker. Simply use `gce_container`
to specify a VM image and a Docker container to execute on the VM (`gce_container` simply extends `gce_instance` definition
with a few additional fields):

```yaml
gce_container:
  image_project: my-project
  image_name: my-custom-ubuntu-with-docker
  container: golang:latest
  additional_containers:
    - name: redis
      image: redis:3.2-alpine
      port: 6379
```

Note that `gce_container` always runs containers in privileged mode.

If your VM image has [Nested Visualization Enabled](https://cloud.google.com/compute/docs/instances/enable-nested-virtualization-vm-instances)
it's possible to use KVM from the container by specifying `enable_nested_virtualization` flag. Here is an example of
using KVM-enabled container to run a hardware accelerated Android emulator:

```yaml
gce_container:
  image_project: my-project
  image_name: my-custom-ubuntu-with-docker-and-KVM
  container: cirrusci/android-sdk:29
  enable_nested_virtualization: true
  accel_check_script:
    - sudo chown cirrus:cirrus /dev/kvm
    - emulator -accel-check
```

#### Instance Scopes

By default Cirrus CI will create Google Compute instances without any [scopes](https://cloud.google.com/sdk/gcloud/reference/alpha/compute/instances/set-scopes) 
so an instance can't access Google Cloud Storage for example. But sometimes it can be useful to give some permissions to an 
instance by using `scopes` key of `gce_instance`.  For example if a particular task builds Docker images and then pushes 
them to [Container Registry](https://cloud.google.com/container-registry/) it's configuration file can look something like:

```yaml
gcp_credentials: ENCRYPTED[qwerty239abc]

gce_instance:
  image_project: my-project
  image_name: my-custom-image-with-docker
  zone: us-central1-a
  cpu: 8
  memory: 40GB
  disk: 20

test_task:
  test_script: ./scripts/test.sh

push_docker_task:
  depends_on: test
  only_if: $CIRRUS_BRANCH == "master"
  gce_instance:
    scopes: cloud-platform
  push_script: ./scripts/push_docker.sh
```

#### Preemptible Instances

Cirrus CI can schedule [preemptible](https://cloud.google.com/compute/docs/instances/preemptible) instances with all price
benefits and stability risks. But sometimes risks of an instance being preempted at any time can be tolerated. For example 
`gce_instance` can be configured to schedule preemptible instance for non master branches like this:

```yaml
gce_instance:
  image_project: my-project
  image_name: my-custom-image-with-docker
  zone: us-central1-a
  preemptible: $CIRRUS_BRANCH != "master"
```

### Kubernetes Engine

<p align="center">
  <a href="#kubernetes-engine">
    <img style="width:128px;height:128px;" src="/assets/images/gcp/Container Engine.svg"/>
  </a>
</p>

Scheduling tasks on [Compute Engine](#google-compute-engine) has one big disadvantage of waiting for an instance to
start which usually takes around a minute. One minute is not that long but can't compete with hundreds of milliseconds
that takes a container cluster on GKE to start a container.

To start scheduling tasks on a container cluster we first need to create one using `gcloud`. Here is a command to create
an auto-scalable cluster that will scale down to zero nodes when there is no load for some time and quickly scale up with
the load during peak hours:

```yaml
gcloud container clusters create cirrus-ci-cluster \
  --zone us-central1-a \
  --num-nodes 1 --machine-type n1-standard-8 \
  --enable-autoscaling --min-nodes=0 --max-nodes=10
```

A service account that Cirrus CI operates via should be assigned with `container.admin` role that allows to administrate GKE clusters:

```bash
gcloud projects add-iam-policy-binding $PROJECT_ID \
    --member serviceAccount:cirrus-ci@$PROJECT_ID.iam.gserviceaccount.com \
    --role roles/container.admin
```

Done! Now after creating `cirrus-ci-cluster` cluster and having `gcp_credentials` configured tasks can be scheduled on the 
newly created cluster like this:

```yaml
gcp_credentials: ENCRYPTED[qwerty239abc]

gke_container:
  image: gradle:jdk8
  cluster_name: cirrus-ci-cluster
  zone: us-central1-a
  namespace: default
  cpu: 6
  memory: 24GB
```

!!! tip "Using in-memory disk"
    By default Cirrus CI mounts a simple [emptyDir](https://kubernetes.io/docs/concepts/storage/volumes/#emptydir) into 
    `/tmp` path to protect the pod from unnecessary eviction by autoscaler. It is possible to switch emptyDir's medium to 
    use in-memory `tmpfs` storage instead of a default one by setting `use_in_memory_disk` field of `gke_container` to `true`
    or any other expression that uses environment variables.

!!! tip "Running privileged containers"
    You can run privileged containers on your private GKE cluster by setting `privileged` field of `gke_container` to `true` 
    or any other expression that uses environment variables. `privileged` field is also available for any additional container.
    
    Here is an example of how to run docker-in-docker
    
    ```yaml hl_lines="9"
    gke_container:
      image: my-docker-client:latest
      cluster_name: my-gke-cluster
      zone: us-west1-c
      namespace: cirrus-ci
      additional_containers:
        - name: docker
          image: docker:dind
          privileged: true
          cpu: 2
          memory: 6G
          port: 2375
    ```

  For a full example on leveraging this to do docker-in-docker builds on Kubernetes checkout [Docker Builds on Kubernetes](docker-builds-on-kubernetes.md)

#### GKE Docker Pipe

It is possible to also launch [Docker Pipes](docker-pipe.md) on your own GKE cluster. Simply use `gke_pipe` instead of `pipe`
and don't forget to specify at least `cluster_name` the same way as for [`gke_container`](#kubernetes-engine). Here is an example
of `.cirrus.yml`:

```yaml
gcp_credentials: ENCRYPTED[qwerty239abc]

gke_pipe:
  cluster_name: my-gke-cluster
  zone: us-west1-c # optional. default is us-central1-a
  namespace: cirrus-ci # optional. default is default

  name: Build Site and Validate Links
  steps:
    - image: squidfunk/mkdocs-material:latest
      build_script: mkdocs build
    - image: raviqqe/liche:latest # links validation tool in a separate container
      validate_script: /liche --document-root=site --recursive site/
```

## AWS

<p align="center">
  <a href="#aws">
    <img style="width:128px;height:128px;" src="/assets/images/aws/AWS.svg"/>
  </a>
  </a>
</p>

Cirrus CI can schedule tasks on several AWS services. In order to interact with AWS APIs Cirrus CI needs permissions. 
Creating an [IMA user](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_users_create.html) for programmatic access 
is a common way to safely give granular access to parts of your AWS.

Once you created a user for Cirrus CI you'll need to provide *key id* and *access key* itself. In order to do so
please create an [encrypted variable](writing-tasks.md#encrypted-variables) with the following content:

```properties
[default]
aws_access_key_id=...
aws_secret_access_key=...
```

Then you'll be able to use the encrypted variable in your `.cirrus.yml` file like this:

```yaml
aws_credentials: ENCRYPTED[...]

task:  
  ec2_instance:
    ...
    
task:  
  eks_instance:
    ...
```

!!! info "Permissions"
    A user that Cirrus CI will be using for orchestrating tasks on AWS should at least have access to S3 in order to store
    logs and cache artifacts. Here is a list of actions that Cirrus CI requires to store logs and artifacts:
    
    ```json
    "Action": [
      "s3:CreateBucket",
      "s3:GetObject",
      "s3:PutObject",
      "s3:DeleteObject",
      "s3:PutLifecycleConfiguration"
    ]
    ```

### EC2

<p align="center">
  <a href="#ec2">
    <img style="width:128px;height:128px;" src="/assets/images/aws/EC2.svg"/>
  </a>
  </a>
</p>

In order to schedule tasks on EC2 please make sure that IAM user that Cirrus CI is using has following permissions:

```json
"Action": [
  "ec2:DescribeInstances",
  "ec2:RunInstances",
  "ec2:TerminateInstances"
]
```

Now tasks can be scheduled on EC2 by configuring `ec2_instance` something like this:

```yaml  
task:
  ec2_instance:
    image: ami-03790f6959fc34ef3
    type: t2.micro
    region: us-east-1
  script: ./run-ci.sh
```

### EKS

<p align="center">
  <a href="#eks">
    <img style="width:128px;height:128px;" src="/assets/images/aws/EKS.svg"/>
  </a>
  </a>
</p>

Please follow instructions on how to [create a EKS cluster](https://docs.aws.amazon.com/eks/latest/userguide/create-cluster.html)
and [add workers nodes](https://docs.aws.amazon.com/eks/latest/userguide/launch-workers.html) to it. And don't forget to
add necessary permissions for the IAM user that Cirrus CI is using:

```json
"Action": [
  "iam:PassRole",
  "eks:DescribeCluster",
  "eks:CreateCluster",
  "eks:DeleteCluster",
  "eks:UpdateClusterVersion"
]
```

To verify that Cirrus CI will be able to communicate with your cluster please make sure that if you are locally logged in
as the user that Cirrus CI acts as you can successfully run the following commands and see your worker nodes up and running:

```bash
$: aws sts get-caller-identity
{
    "UserId": "...",
    "Account": "...",
    "Arn": "USER_USED_BY_CIRRUS_CI"
}
$: aws eks --region $REGION update-kubeconfig --name $CLUSTER_NAME
$: kubectl get nodes
```

!!! tip "EKS Access Denied"
    If you have an issue with accessing your EKS cluster via `kubectl`, most likely you did **not** create the cluster
    with the user that Cirrus CI is using. The easiest way to do so is to create the cluster through AWS CLI with the
    following command:
    
    ```bash
    $: aws sts get-caller-identity
    {
        "UserId": "...",
        "Account": "...",
        "Arn": "USER_USED_BY_CIRRUS_CI"
    }
    $: aws eks --region $REGION \
        create-cluster --name cirrus-ci \
        --role-arn ... \
        --resources-vpc-config subnetIds=...,securityGroupIds=...
    ```   

Now tasks can be scheduled on EKS by configuring `eks_container` something like this:

```yaml  
task:
  eks_container:
    image: node:latest
    region: us-east-1
    cluster_name: cirrus-ci
  script: ./run-ci.sh
```

!!! info "S3 Access for Caching"
    Please add `AmazonS3FullAccess` policy to the role used for creation of EKS workers (same role you put in `aws-auth-cm.yaml`
    when [enabled worker nodes to join the cluster](https://docs.aws.amazon.com/eks/latest/userguide/launch-workers.html)).

## Azure

<p align="center">
  <a href="#google-cloud">
    <img style="width:128px;height:128px;" src="/assets/images/azure/Microsoft Azure.svg"/>
  </a>
  </a>
</p>

Cirrus CI can schedule tasks on several Azure services. In order to interact with Azure APIs 
Cirrus CI needs permissions. First, please choose a subscription you want to use for scheduling CI tasks.
[Navigate to the Subscriptions blade within the Azure Portal](https://portal.azure.com/#blade/Microsoft_Azure_Billing/SubscriptionsBlade)
and save `$SUBSCRIPTION_ID` that we'll use below for setting up a service principle.

Creating a [service principal](https://docs.microsoft.com/en-us/cli/azure/create-an-azure-service-principal-azure-cli) 
is a common way to safely give granular access to parts of Azure:

```bash
az ad sp create-for-rbac --name CirrusCI --sdk-auth \
  --scopes "/subscriptions/$SUBSCRIPTION_ID"
```

Command above will create a new service principal and will print something like:

```json
{
  "appId": "...",
  "displayName": "CirrusCI",
  "name": "http://CirrusCI",
  "password": "...",
  "tenant": "..."
}
``` 

Please create an [encrypted variable](writing-tasks.md#encrypted-variables) from this output and 
add it to the top of `.cirrus.yml` file:

```yaml
azure_credentials: ENCRYPTED[qwerty239abc]
```

Now Cirrus CI can interact with Azure APIs.

### Azure Container Instances

<p align="center">
  <a href="#compute-engine">
    <img style="width:128px;height:128px;" src="/assets/images/azure/Azure Container Service.svg"/>
  </a>
</p>

[Azure Container Instances (ACI)](https://azure.microsoft.com/en-us/services/container-instances/) allows is an ideal 
candidate for running modern CI workloads. ACI allows *just* to run Linux and Windows containers without thinking about 
underlying infrastructure.

Once `azure_credentials` is configured as described above, tasks can be scheduled on ACI by configuring `aci_instance` like this:

```yaml
azure_container_instance:
  image: cirrusci/windowsservercore:2016
  resource_group: CirrusCI
  region: westus
  platform: windows
  cpu: 4
  memory: 12G
```

!!! info "About Docker Images to use with ACI"
    Linux-based images are usually pretty small and doesn't require much tweaking. For Windows containers ACI recommends
    to follow a [few simple advices](https://docs.microsoft.com/en-us/azure/container-instances/container-instances-troubleshooting#container-takes-a-long-time-to-start)
    in order to reduce startup time.

## Anka

<p align="center">
  <a href="#anka">
    <img style="height:100px;" src="/assets/images/veertu/anka-logo.png"/>
  </a>
</p>

[Anka Build by Veertu](https://veertu.com/) is a solution to create private macOS clouds for iOS CI infrastructure.
[Anka Hypervisor](https://veertu.com/anka-technology/#hypervisor) leverages Apple's [`Hypervisor.framework`](https://developer.apple.com/documentation/hypervisor) 
which provides lightweight but powerful macOS VMs that act almost like containers. Overall Anka is a perfect solution 
for a modern Continuous Integration system. 

<p align="center">
  <a href="https://www.macstadium.com/">
    <img style="height:128px;" src="/assets/images/mac-stadium/MacStadium_Logo.png"/>
  </a>
</p>

[MacStadium](https://www.macstadium.com/) is the leading provider of hosted Mac infrastructure and [recently MacStadium partnered with Veertu](https://www.macstadium.com/anka/) 
to provide Hosted Anka Cloud solution. **CI infrastructure for macOS has never been that accessible before.** 

Cirrus CI supports Anka Build as a computing service to schedule tasks on. In order to connect Anka Cloud to Cirrus CI, 
Cirrus Labs created [Anka Controller Extended](https://github.com/cirruslabs/anka-controller-extended) which can connect 
to Anka Cloud's private network and securely expose API for Cirrus CI to connect. 

Please check [Anka Controller Extended Documentation](https://github.com/cirruslabs/anka-controller-extended) for details
and don't hesitate to reach out [support](../support.md) with any question.

Once Anka Controller Extended is up and running, Cirrus CI can use it's API to schedule tasks. Simply use `anka_instance`
in your `.cirrus.yml` file like this: 

```yaml
anka_instance:
  controller_endpoint: <anka-controller-extended-IP>:<PORT>
  access_token: ENCRYPTED[qwerty239]
  template: high-sierra
  tag: xcode-9.4
```

!!! info "Custom Anka VM Templates"
    Anka allows to easily build hierarchy of VMs much like containers with their layers. Please check our [example repository](https://github.com/cirruslabs/osx-images)

!!! info "Hosted Anka Cloud on MacStadium"
    If you choose to use [hosted Anka Cloud solution from MacStadium](https://www.macstadium.com/anka/) please mention
    Cirrus CI upon the registration for a quicker installation process. 
