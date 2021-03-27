---

title: Creating an EKS Cluster
layout: article
image: img/articles/cover/eks.jpg

---

In this tutorial, you’ll learn about

- Installing kubectl and eksctl
- Creating and configuring a EKS cluster


## Installing Kubectl

[Instructions](https://kubernetes.io/docs/tasks/tools/install-kubectl/). Here’s a TLDR for Linux. For other OS, check the link.

Create a folder where you’ll keep it. For example, `~/bin`

Go there, download the kubectl binary:

```bash
curl -LO
https://storage.googleapis.com/kubernetes-release/release/v1.20.0/bin/linux/amd64/kubectl
```

Make it executable:

```bash
chmod +x ./kubectl
```

Add this folder to `PATH`:

```bash
export PATH="~/bin:${PATH}"
```

Add this line to your `.bashrc`


## Install eksctl

Eksctl is a command line tool for creating and managing EKS clusters. [More info](https://docs.aws.amazon.com/eks/latest/userguide/eksctl.html)

Let’s install it to the same `~/.bin` directory where we installed
kubectl:


```bash
curl --silent --location
"https://github.com/weaveworks/eksctl/releases/latest/download/eksctl\_$(uname
-s)_amd64.tar.gz" | tar xz -C ~/bin/
```

## Create a EKS cluster

We’ll use eksctl for creating a cluster. [More info](https://docs.aws.amazon.com/eks/latest/userguide/getting-started-eksctl.html)


```bash
eksctl create cluster \
    --name ml-bookcamp-eks \
    --region eu-west-1
```

Note: if you want to use Fargate, check [this tutorial](https://www.learnaws.org/2019/12/16/running-eks-on-aws-fargate/).
Fargate might be better, but the setup process is more complex.

It should also create a config file in `~/.kube/config`. You should be
able to use kubectl now to connect to the cluster

Check that the config works:

```bash
kubectl get service
```

It should return the list of services currently running on a cluster:

```
NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.100.0.1   <none>        443/TCP   6m17s
```

If you have an error like that:

```
[✖]  unable to use kubectl with the EKS cluster (check 'kubectl version'):
Unable to connect to the server: getting credentials: exec: fork/exec 
/usr/local/bin/aws-iam-authenticator: exec format error
```

You need to generate the config using aws cli ([Instruction](https://docs.aws.amazon.com/eks/latest/userguide/create-kubeconfig.html)).

This is how you do it:

```bash
aws eks --region eu-west-1 update-kubeconfig --name ml-bookcamp-eks
```

It will create a config located at `~/.kube/config`

Check that the config works:

```bash
kubectl get service
```

It should return the list of services currently running on a cluster:

```
NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.100.0.1   <none>        443/TCP   6m17s
```


## Deleting the cluster

Don’t forget to delete your cluster when you finish your experiments.
Use eksctl for that:

```bash
eksctl delete cluster --name ml-bookcamp-eks
```

## Read next

* [Creating a KFServing Cluster on EKS](/article/kfserving-eks-install)
