# EKS Deployment on AWS
The purpose of this simple workshop is to provide guide as a starting of your Kubernetes Journey on top of AWS.
After setting up your EKS Cluster, we will deploy a microservices application.

## 1. Create Cluster VPC using CloudFormation Template

To create your cluster VPC:

   - Open the AWS CloudFormation console at https://console.aws.amazon.com/cloudformation.

   - From the navigation bar, select a Region that supports Amazon EKS.

   - Choose Create stack.

   - For Choose a template, select Specify an Amazon S3 template URL.

   - Paste the following URL into the text area and choose Next: 
    
  ```
  https://amazon-eks.s3-us-west-2.amazonaws.com/cloudformation/2019-02-11/amazon-eks-vpc-sample.yaml
  ```
  
   - On the Specify Details page, fill out the parameters accordingly, and then choose Next.

        Stack name: Choose a stack name for your AWS CloudFormation stack. For example, you can call it eks-vpc.

        VpcBlock: Choose a CIDR range for your VPC. You can keep the default value.

        Subnet01Block: Choose a CIDR range for subnet 1. You can keep the default value.

        Subnet02Block: Choose a CIDR range for subnet 2. You can keep the default value.

        Subnet03Block: Choose a CIDR range for subnet 3. You can keep the default value.

   - (Optional) On the Options page, tag your stack resources. Choose Next.

   - On the Review page, choose Create.

   - When your stack is created, select it in the console and choose Outputs.

   - Record the SecurityGroups value for the security group that was created. You need this when you create your EKS cluster; this security group is applied to the cross-account elastic network interfaces that are created in your subnets that allow the Amazon EKS control plane to communicate with your worker nodes.

   - Record the VpcId for the VPC that was created. You need this when you launch your worker node group template.

   - Record the SubnetIds for the subnets that were created. You need this when you create your EKS cluster; these are the subnets that your worker nodes are launched into.

## 2. Turn on Cloud9 and install kubectl and aws-iam-authenticator

Next step is to prepare a client machine to install kubectl and manage your EKS Cluster/WorkerNodes. We will use Cloud9 for this purpose. Spin up a Cloud9 instance by going to https://ap-southeast-1.console.aws.amazon.com/cloud9/home/product.

**IMPORTANT!** 
Cloud9 rotate credentials (secret/access key) and this is not supported by kubectl, because it detect/match the exact access key that represent IAM User that is used to initially create the cluster.
If you use Cloud9, then ensure you go to > Preferences -> AWS Settings -> Turn off "AWS managed temporary credentials" 
and back to your Cloud9 terminal and run 

```bash
  aws configure
```
to configure your credentials and region where you run EKS Cluster.

Kubernetes uses a command-line utility called kubectl for communicating with the cluster API server. Amazon EKS clusters also require the AWS IAM Authenticator for Kubernetes to allow IAM authentication for your Kubernetes cluster. Beginning with Kubernetes version 1.10, you can configure the kubectl client to work with Amazon EKS by installing the AWS IAM Authenticator for Kubernetes and modifying your kubectl configuration file to use it for authentication. 

**install kubectl**

   ```bash
   mkdir $HOME/bin
   curl -o kubectl https://amazon-eks.s3-us-west-2.amazonaws.com/1.10.3/2018-07-26/bin/linux/amd64/kubectl
   chmod +x ./kubectl
   cp ./kubectl $HOME/bin/kubectl
   export PATH=$HOME/bin:$PATH
   echo 'export PATH=$HOME/bin:$PATH' >> ~/.bashrc
   kubectl version --client
   ```
   
**Install IAM Authenticator**

   ```bash
   curl -o aws-iam-authenticator https://amazon-eks.s3-us-west-2.amazonaws.com/1.10.3/2018-07-26/bin/linux/amd64/aws-iam-authenticator
   chmod +x ./aws-iam-authenticator
   cp ./aws-iam-authenticator $HOME/bin
   export PATH=$HOME/bin:$PATH
   echo 'export PATH=$HOME/bin:$PATH' >> ~/.bashrc
   aws-iam-authenticator help
   ```

## 3. Create your Amazon EKS Cluster

Now you can create your Amazon EKS cluster.

**Important**
When an Amazon EKS cluster is created, the IAM entity (user or role) that creates the cluster is added to the Kubernetes RBAC authorization table as the administrator (with system:master permissions. Initially, only that IAM user can make calls to the Kubernetes API server using kubectl.

To create your cluster with the console

   - Open the Amazon EKS console at https://console.aws.amazon.com/eks/home#/clusters.

   - Choose Create cluster.

   **Note**
   If your IAM user does not have administrative privileges, you must explicitly add permissions for that user to call the  Amazon EKS API operations. For more information, see Creating Amazon EKS IAM Policies.

   - On the Create cluster page, fill in the following fields and then choose Create:

         Cluster name: A unique name for your cluster.

         Kubernetes version: The version of Kubernetes to use for your cluster. By default, the latest available version is selected.

         Role ARN: Select the IAM role that you created with Create your Amazon EKS Service Role.

         VPC: The VPC you created with Create your Amazon EKS Cluster VPC. You can find the name of your VPC in the drop-down list.

         Subnets: The SubnetIds values (comma-separated) from the AWS CloudFormation output that you generated with Create your Amazon EKS Cluster VPC. By default, the available subnets in the above VPC are preselected.

         Security Groups: The SecurityGroups value from the AWS CloudFormation output that you generated with Create your Amazon EKS Cluster VPC. This security group has ControlPlaneSecurityGroup in the drop-down name.

   - On the Clusters page, choose the name of your newly created cluster to view the cluster information.

The Status field shows CREATING until the cluster provisioning process completes. Cluster provisioning usually takes between 10 and 15 minutes.

## 4.Create a KubeConfig file

Use the AWS CLI update-kubeconfig command to create or update your kubeconfig for your cluster.

```
aws eks --region region update-kubeconfig --name cluster_name
```

Test your configuration.

```
kubectl get svc
```

## 5.  Launch and Configure Amazon EKS Worker Nodes 

Wait for your cluster status to show as ACTIVE. If you launch your worker nodes before the cluster is active, the worker nodes will fail to register with the cluster and you will have to relaunch them.

Open the AWS CloudFormation console at https://console.aws.amazon.com/cloudformation.

From the navigation bar, select a Region that supports Amazon EKS.

Choose Create stack.

For Choose a template, select Specify an Amazon S3 template URL.

Paste the following URL into the text area and choose Next:

https://amazon-eks.s3-us-west-2.amazonaws.com/cloudformation/2019-02-11/amazon-eks-nodegroup.yaml

On the Specify Details page, fill out the following parameters accordingly, and choose Next.

   - Stack name: Choose a stack name for your AWS CloudFormation stack. For example, you can call it <cluster-name>-worker-nodes.

   - ClusterName: Enter the name that you used when you created your Amazon EKS cluster.

   **Important**
This name must exactly match the name you used in Step 1: Create Your Amazon EKS Cluster; otherwise, your worker nodes cannot join the cluster.

   - ClusterControlPlaneSecurityGroup: Choose the SecurityGroups value from the AWS CloudFormation output that you generated with Create your Amazon EKS Cluster VPC.

   - NodeGroupName: Enter a name for your node group. This name can be used later to identify the Auto Scaling node group that is created for your worker nodes.

   - NodeAutoScalingGroupMinSize: Enter the minimum number of nodes that your worker node Auto Scaling group can scale in to.

   - NodeAutoScalingGroupDesiredCapacity: Enter the desired number of nodes to scale to when your stack is created.

   - NodeAutoScalingGroupMaxSize: Enter the maximum number of nodes that your worker node Auto Scaling group can scale out to.

   - NodeInstanceType: Choose an instance type for your worker nodes.

   - NodeImageId: Enter the current Amazon EKS worker node AMI ID for your Region (ami-07b922b9b94d9a6d2). 
   
## 6. Create AWS authenticator configuration map

Download, edit, and apply the AWS authenticator configuration map:

Download the configuration map with the following command:

    curl -o aws-auth-cm.yaml https://amazon-eks.s3-us-west-2.amazonaws.com/cloudformation/2019-02-11/aws-auth-cm.yaml

Open the file with your favorite text editor. Replace the <ARN of instance role (not instance profile)> snippet with the NodeInstanceRole value that you recorded in the previous procedure, and save the file.

    Important

    Do not modify any other lines in this file.

    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: aws-auth
      namespace: kube-system
    data:
      mapRoles: |
        - rolearn: <ARN of instance role (not instance profile)>
          username: system:node:{{EC2PrivateDNSName}}
          groups:
            - system:bootstrappers
            - system:nodes

Apply the configuration. This command might take a few minutes to finish.

    kubectl apply -f aws-auth-cm.yaml

Watch the status of your nodes and wait for them to reach the Ready status.

    kubectl get nodes --watch


