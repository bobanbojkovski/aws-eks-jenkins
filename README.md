# AWS EKS &amp; Jenkins

Motivation of the following sample is to setup Jenkins on EKS cluster and continue with EKS autoscaling exploration tasks.



### Install required tools 

Here we use [eksctl](https://eksctl.io/) to manage the AWS EKS, [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/) to work with the Kubernetes, [helm](https://helm.sh/docs/intro/install/) to manage the Kubernetes applications.


### AWS VPC Configuration

Reuse/configure manually the networking for the EKS cluster.

```
Create/use existing VPC.
Set VPC name and CIDR.
Enable DNS hostnames & DNS resolution.

Create Subnets, public and private for each AZ in selected region.

Create Internet Gateway and attach it to the VPC.

Create NAT Gateway in public subnet/s and select elastic IP.

Set routing tables for each subnet, public subnets route to IGW and private subnets route to the NAT GW.

Associate the routing tables with the public and private subnets.
```


### Deploy EKS Cluster

Define EKS Cluster configurations in a yaml file, for example  
jenkins-eks.yaml:

```
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: <jenkins-cluster>
  region: <region>
  version: "1.17"

vpc:
  cidr: <cidr>
  id: <vpc-id>
  subnets:
    private:
      <private-subnet-name-1>:
        id: <private-subnet-1-id>
      <private-subnet-name-2>:
        id: <private-subnet-2-id>
      <private-subnet-name-3>:
        id: <private-subnet-3-id>
    public:
      <public-subnet-name-1>:
        id: <public-subnet-1-id>
      <public-subnet-name-2>:
        id: <public-subnet-2-id>
      <public-subnet-name-3>:
        id: <public-subnet-3-id>

managedNodeGroups:
  - name: ng-jenkins-master-ondemand
    instanceType: t3.medium
    desiredCapacity: 1
    privateNetworking: true
    subnets:
      - <private-subnet-name-1>
    
  - name: ng-jenkins-agents-spot
    instanceTypes: ["t3.small", "t3.medium"]
    spot: true
    minSize: 1
    maxSize: 2
    desiredCapacity: 1
    privateNetworking: true

    iam:
      withAddonPolicies:
        autoScaler: true
```

Deploy the jenkins-cluster defined in jenkins-eks.yaml.
```
AWS_PROFILE=<aws-profile> eksctl create cluster -f jenkins-eks.yaml
```
  
The eksctl uses CloudFormation to create the EKS Cluster resources.  
After the cluster is created check the stack, and cluster configuration.  
```
AWS_PROFILE=<aws-profile> eksctl utils describe-stacks --region=<region> --cluster=<jenkins-cluster>
AWS_PROFILE=<aws-profile> aws eks --region <region> describe-cluster --name <jenkins-cluster>
AWS_PROFILE=<aws-profile> kubectl cluster-info
AWS_PROFILE=<aws-profile> kubectl get nodes -o wide
```
  
To destroy the cluster run `delete cluster` command.
```
AWS_PROFILE=<aws-profile> eksctl delete cluster -f jenkins-eks.yaml
```


### Install Kubernete metrics server
Follow instructions on [Installing the Kubernetes Metrics Server](https://docs.aws.amazon.com/eks/latest/userguide/metrics-server.html)  
Check for the latest release at https://github.com/kubernetes-sigs/metrics-server/


### Install Kubernetes dashboard
Follow instructions on [Tutorial: Deploy the Kubernetes Dashboard](https://docs.aws.amazon.com/eks/latest/userguide/dashboard-tutorial.html)  
To delete the kubenetes resources run, `kubectl delete -f <yaml-file>` command.


### Deploy AWS EFS for Jenkins

Create a security group for the EFS mount target.

```
AWS_PROFILE=<aws-profile> aws ec2 create-security-group \
--region <region> \
--group-name efs-jenkins-sg \
--description "EFS for EKS Jenkins" \
--vpc-id <vpc-id>

Save the output, GroupId.
```
[create-security-group](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/ec2/create-security-group.html)

To delete security group use `delete-security-group` command.  
```
AWS_PROFILE=<aws-profile> aws ec2 delete-security-group \
--group-id <GroupId>
```
[delete-security-group](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/ec2/delete-security-group.html)

Specify ingress rules to the security group.
```
AWS_PROFILE=<aws-profile> aws ec2 authorize-security-group-ingress \
--group-id <GroupId> \
--region <region> \
--protocol tcp \
--port 2049 \
--cidr <vpc-cidr>
```
[authorize-security-group-ingress](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/ec2/authorize-security-group-ingress.html)

Create empty EFS file system.

```
AWS_PROFILE=<aws-profile> aws efs create-file-system \
--creation-token eks-jenkins-efs \
--performance-mode generalPurpose \
--throughput-mode bursting \
--region <region> \
--tags Key=Name,Value=JenkinsEFS \
--encrypted

Save the output, FileSystemId
```
[create-file-system](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/efs/create-file-system.html)

To delete the specified file system use, `delete-file-system` command.
```
AWS_PROFILE=<aws-profile> aws efs delete-file-system \
--file-system-id <FileSystemId>
```
[delete-file-system](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/efs/delete-file-system.html)

Create one mount target in each Availability Zone for the EFS file system.

```
AWS_PROFILE=<aws-profile> aws efs create-mount-target \
--file-system-id <FileSystemId> \
--subnet-id <private-subnet-name-1> \
--security-group <GroupId> \
--region <region>

AWS_PROFILE=<aws-profile> aws efs create-mount-target \
--file-system-id <FileSystemId> \
--subnet-id <private-subnet-name-2> \
--security-group <GroupId> \
--region <region>

AWS_PROFILE=<aws-profile> aws efs create-mount-target \
--file-system-id <FileSystemId> \
--subnet-id <private-subnet-name-3> \
--security-group <GroupId> \
--region <region>
```
[create-mount-target](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/efs/create-mount-target.html)

To delete the mount targets use `delete-mount-target` command.
```
AWS_PROFILE=<aws-profile> aws efs delete-mount-target \
--mount-target-id <MountTargetId>
```
[delete-mount-target](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/efs/delete-mount-target.html)

Create an access point for the EFS file system.
```
AWS_PROFILE=<aws-profile> aws efs create-access-point \
--file-system-id <FileSystemId> \
--posix-user Uid=1000,Gid=1000 \
--root-directory "Path=/jenkins,CreationInfo={OwnerUid=1000,OwnerGid=1000,Permissions=777}" \
--tags Key=Name,Value=JenkinsAP \
--region <region>

Save the output, AccessPointId.
```
[create-access-point](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/efs/create-access-point.html)

To delete the access point use, `delete-access-point` command.
```
AWS_PROFILE=<aws-profile> aws efs delete-access-point \
--access-point-id <AccessPointId>
```
[delete-access-point](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/efs/delete-access-point.html)


### Deploy the AWS EFS CSI driver to the EKS cluster

[Amazon EFS CSI Driver](https://github.com/kubernetes-sigs/aws-efs-csi-driver)

```
AWS_PROFILE=<aws-profile> kubectl apply -k "github.com/kubernetes-sigs/aws-efs-csi-driver/deploy/kubernetes/overlays/stable/?ref=release-1.0"
```

To delete the EFS CSI Driver run `kubectl delete` command.
```
AWS_PROFILE=<aws-profile> kubectl delete -k "github.com/kubernetes-sigs/aws-efs-csi-driver/deploy/kubernetes/overlays/stable/?ref=release-1.0"
```


### Create Kubernetes namespace for Jenkins application

```
cat << EOF > jenkins-namespace.yaml
---
apiVersion: v1
kind: Namespace
metadata:
  name: jenkins
EOF
```

Deploy the namespace.
```
AWS_PROFILE=<aws-profile> kubectl apply -f jenkins-namespace.yaml
```

To delete the `jenkins` namespace.
```
AWS_PROFILE=<aws-profile> kubectl delete -f jenkins-namespace.yaml
```


### Deploy Kubernetes storage resources

Storage class configuration.
```
cat << EOF > storage-class.yaml
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: efs-sc
  namespace: jenkins
provisioner: efs.csi.aws.com
EOF
```

Deploy the storage class configuration.
```
AWS_PROFILE=<aws-profile> kubectl apply -f storage-class.yaml
```

Persistent volume configuration, set **FileSystemId::AccessPointId** to `volumeHandle` value.
```
cat << EOF > persistent-volume.yaml
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: efs-pv
  namespace: jenkins
spec:
  capacity:
    storage: 8Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  storageClassName: efs-sc
  csi:
    driver: efs.csi.aws.com
    volumeHandle: <FileSystemId>::<AccessPointId>
EOF
```

Deploy the persistent volume configuration.
```
AWS_PROFILE=<aws-profile> kubectl apply -f persistent-volume.yaml
```

Persistent volume claim
```
cat << EOF > persistent-volume-claim.yaml
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: efs-claim
  namespace: jenkins
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: efs-sc
  resources:
    requests:
      storage: 8Gi
EOF
```

Deploy the persistent volume claim configuration.
```
AWS_PROFILE=<aws-profile> kubectl apply -f persistent-volume-claim.yaml
```

Check the deployed storage resources states.
```
AWS_PROFILE=<aws-profile> kubectl get sc,pv,pvc -n jenkins
```

To delete the storage resources run following command.
```
AWS_PROFILE=<aws-profile> kubectl delete -f storage-class.yaml,persistent-volume.yaml,persistent-volume-claim.yaml
```


### Load Balancer Controller Installation

[Load Balancer Controller Installation](https://kubernetes-sigs.github.io/aws-load-balancer-controller/latest/deploy/installation/)

Create IAM OIDC provider
```
AWS_PROFILE=<aws-profile> eksctl utils associate-iam-oidc-provider \
--region <region> \
--cluster <jenkins-cluster> \
--approve
```
[IAM Roles for Service Accounts](https://eksctl.io/usage/iamserviceaccounts/)

To delete the oidc provider use aws delete-open-id-connect-provider command.
```
AWS_PROFILE=<aws-profile> aws eks describe-cluster \
--region <region> \
--name <jenkins-cluster> \
--query cluster.identity.oidc.issuer \
--output text

Use the output in the following command

AWS_PROFILE=<aws-profile> aws iam delete-open-id-connect-provider \
--open-id-connect-provider-arn arn:aws:iam::<aws-account-id>:oidc-provider/<the-previous-command-output>
```
[describe-cluster](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/eks/describe-cluster.html)  
[delete-open-id-connect-provider](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/iam/delete-open-id-connect-provider.html)

Download IAM policy for the AWS Load Balancer Controller.
```
curl -o iam-policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.1.0/docs/install/iam_policy.json
```

Create an IAM policy called AWSLoadBalancerControllerIAMPolicy
```
AWS_PROFILE=<aws-profile> aws iam create-policy \
--policy-name AWSLoadBalancerControllerIAMPolicy \
--policy-document file://iam-policy.json
```
[create-policy](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/iam/create-policy.html)

To delete the policy use delete-policy command.
```
AWS_PROFILE=<aws-profile> aws iam delete-policy \
--policy-arn arn:aws:iam::<aws-account-id>:policy/AWSLoadBalancerControllerIAMPolicy
```
[delete-policy](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/iam/delete-policy.html)

Create a IAM role and ServiceAccount for the AWS Load Balancer controller.
```
AWS_PROFILE=<aws-profile> eksctl create iamserviceaccount \
--region <region> \
--cluster=<jenkins-cluster> \
--namespace=kube-system \
--name=aws-load-balancer-controller \
--attach-policy-arn=arn:aws:iam::<aws-account-id>:policy/AWSLoadBalancerControllerIAMPolicy \
--approve
```
[IAM Roles for Service Accounts](https://eksctl.io/usage/iamserviceaccounts/)

Delete the IAM role and serviceaccount.
```
AWS_PROFILE=sandbox eksctl delete iamserviceaccount \
--region <region> \
--cluster=<jenkins-cluster> \
--namespace=kube-system \
--name=aws-load-balancer-controller
```
[IAM Roles for Service Accounts](https://eksctl.io/usage/iamserviceaccounts/)


#### Add the controller to EKS cluster.

Add the EKS chart repo to helm.
```
helm repo add eks https://aws.github.io/eks-charts
```

Install the TargetGroupBinding CRDs.
```
AWS_PROFILE=<aws-profile> kubectl apply -k "github.com/aws/eks-charts/stable/aws-load-balancer-controller//crds?ref=master"
```

To uninstall the TargetGroupBinding CRDs.
```
AWS_PROFILE=<aws-profile> kubectl delete -k "github.com/aws/eks-charts/stable/aws-load-balancer-controller//crds?ref=master"
```

Install the helm chart.
```
AWS_PROFILE=<aws-profile> helm upgrade -i aws-load-balancer-controller \
eks/aws-load-balancer-controller \
-n kube-system \
--set image.repository=amazon/aws-alb-ingress-controller \
--set clusterName=<jenkins-cluster> \
--set serviceAccount.create=false \
--set serviceAccount.name=aws-load-balancer-controller \
--set region=<region> \
--set vpcId=<vpc-id>
```

To uninstall the aws-load-balancer-controller resource run,
```
AWS_PROFILE=<aws-profile> helm uninstall aws-load-balancer-controller -n kube-system
```

Create subnets tags, Key/Value.
```
Public subnets tag
kubernetes.io/role/elb      1

Private submnets tag
kubernetes.io/role/internal-elb     1
```
[Application load balancing on Amazon EKS](https://docs.aws.amazon.com/eks/latest/userguide/alb-ingress.html)


### Deploy Jenkins application

reference [jenkinsci/helm-charts](https://github.com/jenkinsci/helm-charts/blob/main/charts/jenkins/README.md)

Add the Jenkins chart repository
```
helm repo add jenkins https://charts.jenkins.io
```

Generate `values.yaml` file.
```
helm show values jenkins/jenkins > values.yaml
```

Update the `values.yaml` with your configuration.  
For ingress setup check documentation on [AWS LoadBalancer Controller](https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.1/)

Deploy the updated `values.yaml` configuration in `jenkins` namespace.
```
AWS_PROFILE=<aws-profile> helm install jenkins -n jenkins jenkins/jenkins -f values.yaml
```

Verify the Jenkins resources are created.
```
AWS_PROFILE=<aws-profile> kubectl get all -n jenkins
AWS_PROFILE=<aws-profile> kubectl get ingress -n jenkins
```

To uninstall Jenkins, run `helm uninstall` command.
```
AWS_PROFILE=<aws-profile> helm uninstall jenkins -n jenkins
```


### Configure Route 53 record

Manual steps to update the Route 53 for Jenkins application.
```
Select a hosted zones.

For record name use the jenkins hostname defined in the values.yaml file.

Define route traffic to ELB.

Specify A record type.

Use simple routing policy.

Enable evaluate target health.
```
[Configuring Amazon Route 53 as your DNS service](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/dns-configuring.html)
  
  
Better alternative, use [externalDNS](https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.1/guide/integrations/external_dns/) to setup and manage records in Route 53.

Download [ExternalDNS IAM policy](https://github.com/kubernetes-sigs/external-dns/blob/master/docs/tutorials/aws.md#iam-permissions) and create AWSExternalDNSIAMPolicy.

external-dns-policy.json
```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "route53:ChangeResourceRecordSets"
      ],
      "Resource": [
        "arn:aws:route53:::hostedzone/*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "route53:ListHostedZones",
        "route53:ListResourceRecordSets"
      ],
      "Resource": [
        "*"
      ]
    }
  ]
}
```

Deploy the external-dns policy.
```
AWS_PROFILE=<aws-profile> aws iam create-policy \
--policy-name AWSExternalDNSIAMPolicy \
--policy-document file://external-dns-policy.json

Save the output, Arn
```

Create IAM Role, k8s Service Account & Associate IAM Policy.
```
AWS_PROFILE=<aws-profile> eksctl create iamserviceaccount \
  --region <region> \
  --cluster <jenkins-cluster> \
  --namespace kube-system \
  --name aws-external-dns \
  --attach-policy-arn arn:aws:iam::<aws-account-id>:policy/AWSExternalDNSIAMPolicy \
  --approve 
```

Install external DNS  
wget https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.0.0/docs/examples/external-dns.yaml

Edit the external-dns.yaml file.  
Add annotation for iamserviceaccount role created in previous step.
```
annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam:::<aws-account-id>:role/eksctl-jenkins-test-addon-iamserviceaccount-Role1-1GF8IUCBW6KM1
```
also, edit the arguments for Route53.
```
- --domain-filter # name of hosted zone, example.com
- --aws-zone-type # public or private zone, empty means both zones
# in case of txt record
- --txt-owner-id # hosted zone id
```

Edit the ClusterRoleBinding namespace to kube-system instead of default.

Deploy the external-dns configuration.
```
AWS_PROFILE=<aws-profile> kubectl apply -f external-dns.yaml -n kube-system
```

### Proceed with AWS Autoscaler

[Cluster Autoscaler](https://docs.aws.amazon.com/eks/latest/userguide/cluster-autoscaler.html)


