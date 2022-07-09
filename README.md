# eks-demo

This was created and tested on 7/9/2022

## Initialize a VM using Vagrant

The Vagrant file here is using the default `ubuntu/jammy64` image. This is Ubuntu 22.04 LTS.

* `vagrant up` - Initialize VM - This takes a little bit of time to complete. Might be worth jumping to Create IAM User section while you wait.
* `vagrant ssh` - Connect to VM

## Install dependencies

```bash
sudo apt-get update
sudo apt-get -y install unzip
```


## Create IAM User / Give Permissions

  * I created a user called `eks-test-user`
  * Generate Access Key 
    * Take note of the Access Key ID and Secret Access Key you will need these later
  * I created a Group called "EKS_Full"
      * I Included all default EKS permissions provided by amazon.
        * AmazonEKSVPCResourceController
        * AmazonEKSFargatePodExecutionRolePolicy
        * AmazonEKS_CNI_Policy
        * AmazonEKSServicePolicy
        * AmazonEKSWorkerNodePolicy
        * AmazonEKSClusterPolicy
      * This still wasn't enough so I ended up creating an EKSAdministrator Permission: [eksadministrator.json](eksadministrator.json)
        * Note this feels like a bad idea. It would be better to figure out the exact resources required
        * It may be possible to use just this permission without those listed above.
      * I also needed `EC2FullAccess` -
      * I also needed `AWSCloudFormationFullAccess` - 
      * I also needed `IAMFullAccess` - This one seems extreme. But during the process it needed to create roles for the cluster. It is probably possible to trim this one down. 
      

## Install Amazon version of kubectl

Amazon has their own special version of kubectl. It can be installed with the following commands:

```bash
curl -o kubectl https://s3.us-west-2.amazonaws.com/amazon-eks/1.22.6/2022-03-09/bin/linux/amd64/kubectl
* `chmod +x ./kubectl
* `mkdir -p $HOME/bin && cp ./kubectl $HOME/bin/kubectl && export PATH=$PATH:$HOME/bin
* `echo 'export PATH=$PATH:$HOME/bin' >> ~/.bashrc
```

Verify:
```
vagrant@ubuntu-jammy:~$ date
Sat Jul  9 13:46:35 UTC 2022
vagrant@ubuntu-jammy:~$ kubectl version --short
Client Version: v1.22.6-eks-7d68063
The connection to the server localhost:8080 was refused - did you specify the right host or port?
```

## Install aws cli

```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
```

Verify:
```
vagrant@ubuntu-jammy:~$ date
Sat Jul  9 13:46:35 UTC 2022
vagrant@ubuntu-jammy:~$ aws --version
aws-cli/2.7.14 Python/3.9.11 Linux/5.15.0-40-generic exe/x86_64.ubuntu.22 prompt/off
```

## Configure aws cli

`aws configure`

## Install eksctl
This is a command line tool for managing EKS clusters. Documentation here: [https://eksctl.io/](https://eksctl.io/)

```bash
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin
eksctl version
```

Verify:
```
vagrant@ubuntu-jammy:~$ date 
Sat Jul  9 13:48:31 UTC 2022
vagrant@ubuntu-jammy:~$ eksctl version
0.105.0
vagrant@ubunt
```

## Create an EKS Cluster

```
CLUSTER_NAME="Test-Cluster"
REGION="us-east-1"
eksctl create cluster --name "$CLUSTER_NAME" --region "$REGION"
```

This created a Stack on CloudFormation. It is probably an important task to learn more about the different flags that can be used with this utility
to be able to modify the stack, start it, and stop it. 

This took several minutes but you can watch it build in CloudFormation in the Events tab.
The longest part was the actual creation of the EKS cluster itself. You can see it being created in the EKS console.

This looks almost identical to same layout as we have seen in the past: 2 Public Subnets and 2 Private Subnets.
The template used is here: [cloud-formation-template.json](cloud-formation-template.json)

The command creates the cluster in the following way:
* It does not include any Add-ons
* It includes a Nodegroup with 2 nodes by default. These nodes are set to be `m5.large` instances.

### Config file / Command line arguments
You can specify a configuration file for how the cluster should be created. For example: [eks-example-cluster.yaml](eks-example-cluster.yaml)

Below does not work:
You can also specify the node groups from the command line:

```
CLUSTER_NAME="Test-Cluster"
REGION="us-east-1"
NODE_COUNT=2 # Desired Count
NODE_VOLUME_SIZE=20 # Size in GB
NODE_VOLUME_TYPE="t3.medium" # Note these need to be big enough to run docker containers
eksctl create cluster \
   --name "$CLUSTER_NAME" \
   --region "$REGION" \
   --nodes="$NODE_COUNT" \
   --node-volume-size="$NODE_VOLUME_SIZE" \
   --node-volume-type="$NODE_VOLUME_TYPE"
```

This command also updates the ~/.kube folder and adds the cluster to the config mapping.

## Examine the Cluster

The previous command updated the configuration. We can run `kubectl` commands to examine the cluster

* `kubectl get nodes -o wide`
* `kubectl get pods -A -o wide`

## Deploy Some example pods

I took an example deployment from [https://kubernetes.io/docs/concepts/workloads/controllers/deployment/](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/) and placed it in [nginx-deployment.yaml]

You can copy it into the vagrant home directory using: `cp /vagrant/nginx-deployment.yaml .`
Then, we can create a deployment by running:

```
kubectl apply -f nginx-deployment.yaml
```

We can then check the deployment:

```
kubectl get deployments
```

## Deleting the Cluster (Save some $$)

```
CLUSTER_NAME="Test-Cluster"
REGION="us-east-1"
eksctl delete cluster --name "$CLUSTER_NAME" --region "$REGION"
```

## Shutting Down Vagrant

Exit the vagrant: `exit`

Then run: `vagrant halt`
You can come back later with `vagrant up` and then `vagrant ssh`
You can destroy the whole VM using `vagrant destroy`. If you do this you will need to re-run the entire installation including the aws configure.