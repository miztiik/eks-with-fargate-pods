# Kubernetes Scheduling: Node Selection, Affinity & AntiAffinity

The developer at Mystique Unicorn are interested in building their application using event-driven architectural pattern to process streaming data. For those who are unfamiliar, _An event-driven architecture uses events to trigger and communicate between decoupled services and is common in modern applications built with microservices. An event is a change in state, or an update, like an item being placed in a shopping cart on an e-commerce website._

In this application, Kubernetes has been chosen as the platform to host their application producing and consuming events. The developers do not want to worry about patching, scaling, or securing a cluster of EC2 instances to run Kubernetes applications in the cloud. They are looking for a low-overhead mechnaism to run their pods

Can you help them?

![Miztiik Automation: Kubernetes Scheduling: Node Selection, Affinity & AntiAffinity](images/miztiik_automation_eks_scheduling_with_affinity_architecture_000.png)

## 🎯 Solutions

AWS Fargate<sup>[1]</sup> is a serverless compute engine for containers that works with Amazon Elastic Kubernetes Service (EKS). Fargate reliminates the need for customers to create or manage EC2 instances for their Amazon EKS clusters. Using Fargate, customers define and pay for resources at the pod-level. This makes it easy to right-size resource utilization for each application and allow customers to clearly see the cost of each pod.

Fargate allocates the right amount of compute, eliminating the need to choose instances and scale cluster capacity. You only pay for the resources required to run your containers, so there is no over-provisioning and paying for additional servers. Fargate runs each task or pod in its own kernel providing the tasks and pods their own isolated compute environment. This enables your application to have workload isolation and improved security by design.

In this blog, I will show how to deploy a simple application using Amazon EKS on Fargate.

1. ## 🧰 Prerequisites

   This demo, instructions, scripts and cloudformation template is designed to be run in `us-east-1`. With few modifications you can try it out in other regions as well(_Not covered here_).

   - 🛠 AWS CLI Installed & Configured - [Get help here](https://youtu.be/TPyyfmQte0U)
   - 🛠 AWS CDK Installed & Configured - [Get help here](https://www.youtube.com/watch?v=MKwxpszw0Rc)
   - 🛠 Python Packages, _Change the below commands to suit your OS, the following is written for amzn linux 2_
     - Python3 - `yum install -y python3`
     - Python Pip - `yum install -y python-pip`
     - Virtualenv - `pip3 install virtualenv`

1. ## ⚙️ Setting up the environment

   - Get the application code

     ```bash
     git clone https://github.com/miztiik/eks-security-with-psp
     cd eks-security-with-psp
     ```

1. ## 🚀 Prepare the dev environment to run AWS CDK

   We will use `cdk` to make our deployments easier. Lets go ahead and install the necessary components.

   ```bash
   # You should have npm pre-installed
   # If you DONT have cdk installed
   npm install -g aws-cdk

   # Make sure you in root directory
   python3 -m venv .venv
   source .venv/bin/activate
   pip3 install -r requirements.txt
   ```

   The very first time you deploy an AWS CDK app into an environment _(account/region)_, you’ll need to install a `bootstrap stack`, Otherwise just go ahead and deploy using `cdk deploy`.

   ```bash
   cdk bootstrap
   cdk ls
   # Follow on screen prompts
   ```

   You should see an output of the available stacks,

   ```bash
   eks-cluster-vpc-stack
   eks-cluster-stack
   ssm-agent-installer-daemonset-stack
   ```

1. ## 🚀 Deploying the application

   Let us walk through each of the stacks,

   - **Stack: eks-cluster-vpc-stack**
     To host our EKS cluster we need a custom VPC. This stack will build a multi-az VPC with the following attributes,

     - **VPC**:
       - 2-AZ Subnets with Public, Private and Isolated Subnets.
       - 1 NAT GW for internet access from private subnets

     Initiate the deployment with the following command,

     ```bash
     cdk deploy eks-cluster-vpc-stack
     ```

     After successfully deploying the stack, Check the `Outputs` section of the stack.

   - **Stack: eks-cluster-stack**
     As we are starting out a new cluster, we will use most default. No logging is configured or any add-ons. The cluster will have the following attributes,

     - The control pane is launched with public access. _i.e_ the cluster can be access without a bastion host
     - `c_admin` IAM role added to _aws-auth_ configMap to administer the cluster from CLI.
     - One **OnDemand** managed EC2 node group created from a launch template
       - It create two `t3.medium` instances running `Amazon Linux 2`
       - Auto-scaling Group with `2` desired instances.
       - The nodes will have a node role attached to them with `AmazonSSMManagedInstanceCore` permissions
       - Kubernetes label `app:miztiik_on_demand_ng`
     - One **Spot** managed EC2 node group created from a launch template
       - It create two `t3.large` instances running `Amazon Linux 2`
       - Auto-scaling Group with `1` desired instances.
       - The nodes will have a node role attached to them with `AmazonSSMManagedInstanceCore` permissions

     In this demo, let us launch the EKS cluster in a custom VPC using AWS CDK. Initiate the deployment with the following command,

     ```bash
     cdk deploy eks-cluster-stack
     ```

     After successfully deploying the stack, Check the `Outputs` section of the stack. You will find the `*ConfigCommand*` that allows yous to interact with your cluster using `kubectl`

   - **Stack: ssm-agent-installer-daemonset-stack**
     This EKS AMI used in this stack does not include the AWS SSM Agent out of the box. If we ever want to patch or run something remotely on our EKS nodes, this agent is really helpful to automate those tasks. We will deploy a daemonset that will _run exactly once?_ on each node using a cron entry injection that deletes itself after successful execution. If you are interested take a look at the deamonset manifest here `stacks/back_end/eks_cluster_stacks/eks_ssm_daemonset_stack/eks_ssm_daemonset_stack.py`. This is inspired by this AWS guidance.

     Initiate the deployment with the following command,

     ```bash
     cdk deploy ssm-agent-installer-daemonset-stack
     ```

     After successfully deploying the stack, You can use connect to the worker nodes instance using SSM Session Manager.

1. ## 🔬 Testing the solution

   We are all set with our cluster to deploy our pods.

   1. **Create Producer Pods**

      Since this is demo is about `nodeSelector` and _affinity_ feature of kubernetes, We will run _busybox_ image and label it `miztiik-producer`. We will later use this label to constraint our consumers.

      I have included a sample manifest here `stacks/k8s_utils/sample_manifests/producer_anti_affinity.yml`. The interesting thing to note here is the node selector label `miztiik_on_demand_ng`. This will ensure the deployment runs only on ondemand instances. We are also using the `podAntiAffinity` to ensure that none of the producers are placed alongside consumer pods.

      ```text
      apiVersion: apps/v1
      kind: Deployment
      metadata:
        name: on-demand-producers
      spec:
        selector:
          matchLabels:
            app: miztiik-producer
        replicas: 3
        template:
          metadata:
            labels:
              app: miztiik-producer
          spec:
            affinity:
              podAntiAffinity:
                requiredDuringSchedulingIgnoredDuringExecution:
                - labelSelector:
                    matchExpressions:
                    - key: app
                      operator: In
                      values:
                      - miztiik-consumer
                  topologyKey: "kubernetes.io/hostname"
            nodeSelector:
              app: miztiik_on_demand_ng
            containers:
            - name: busybox
              image: busybox
              command: [ "sh", "-c", "sleep 10h" ]
      ```

      Deploy the manifest,

      ```bash
      kubectl get po --selector app=miztiik-producer

      ```

      Expected output,

      ```bash
      NAME                                   READY   STATUS    RESTARTS   AGE
      on-demand-producers-5c7d4dfcc9-dmv4c   1/1     Running   2          26h
      on-demand-producers-5c7d4dfcc9-msssw   1/1     Running   2          26h
      on-demand-producers-5c7d4dfcc9-vcxfz   1/1     Running   2          26h
      ```

   1. **Create Consumers**:

      Since this is demo is about `nodeSelector` and _affinity_ feature of kubernetes, We will run _busybox_ image and label it `miztiik-consumer`. We will later use this label to constraint our consumers.

      I have included a sample manifest here `stacks/k8s_utils/sample_manifests/consumer_anti_affinity.yml`. The interesting thing to note here is the node selector label `miztiik_spot_ng`. This will ensure the deployment runs only on spot instances. We are also using the `podAntiAffinity` to ensure that none of the consumers are placed alongside producer pods.

      ```text
      apiVersion: apps/v1
      kind: Deployment
      metadata:
        name: spot-consumers
      spec:
        selector:
          matchLabels:
            app: miztiik-consumer
        replicas: 3
        template:
          metadata:
            labels:
              app: miztiik-consumer
          spec:
            affinity:
              podAntiAffinity:
                requiredDuringSchedulingIgnoredDuringExecution:
                - labelSelector:
                    matchExpressions:
                    - key: app
                      operator: In
                      values:
                      - miztiik-producer
                  topologyKey: "kubernetes.io/hostname"
            nodeSelector:
              app: miztiik_spot_ng
            containers:
            - name: busybox
              image: busybox
              command: [ "sh", "-c", "sleep 10h" ]
      ```

      Deploy the manifest,

      ```bash
      kubectl get po --selector app=miztiik-consumer

      ```

      Expected output,

      ```bash
      NAME                              READY   STATUS    RESTARTS   AGE
      spot-consumers-6cb6bd49dd-8kxwt   1/1     Running   2          26h
      spot-consumers-6cb6bd49dd-lc4rd   1/1     Running   2          26h
      spot-consumers-6cb6bd49dd-lpshs   1/1     Running   2          26h
      ```

1. ## 📒 Conclusion

   Here we have demonstrated how to use Kubernetes selectors to schedule pods. Given the complexities involved these features need to be used carefully.

1. ## 🧹 CleanUp

   If you want to destroy all the resources created by the stack, Execute the below command to delete the stack, or _you can delete the stack from console as well_

   - Resources created during [Deploying The Application](#-deploying-the-application)
   - Delete CloudWatch Lambda LogGroups
   - _Any other custom resources, you have created for this demo_

   ```bash
   # Delete from cdk
   cdk destroy

   # Follow any on-screen prompts

   # Delete the CF Stack, If you used cloudformation to deploy the stack.
   aws cloudformation delete-stack \
     --stack-name "MiztiikAutomationStack" \
     --region "${AWS_REGION}"
   ```

   This is not an exhaustive list, please carry out other necessary steps as maybe applicable to your needs.

## 📌 Who is using this

This repository aims to show how to schedule pods using kubernetes schedulers to new developers, Solution Architects & Ops Engineers in AWS. Based on that knowledge these Udemy [course #1][102], [course #2][101] helps you build complete architecture in AWS.

### 💡 Help/Suggestions or 🐛 Bugs

Thank you for your interest in contributing to our project. Whether it is a bug report, new feature, correction, or additional documentation or solutions, we greatly value feedback and contributions from our community. [Start here](/issues)

### 👋 Buy me a coffee

[![ko-fi](https://www.ko-fi.com/img/githubbutton_sm.svg)](https://ko-fi.com/Q5Q41QDGK) Buy me a [coffee ☕][900].

### 📚 References

1. [AWS Docs: Fargate][1]
1. [Kubernetes Docs: Assigning Pods to Nodes][2]

### 🏷️ Metadata

![miztiik-success-green](https://img.shields.io/badge/Miztiik:Automation:Level-200-blue)

**Level**: 200

[1]: https://aws.amazon.com/fargate
[100]: https://www.udemy.com/course/aws-cloud-security/?referralCode=B7F1B6C78B45ADAF77A9
[101]: https://www.udemy.com/course/aws-cloud-security-proactive-way/?referralCode=71DC542AD4481309A441
[102]: https://www.udemy.com/course/aws-cloud-development-kit-from-beginner-to-professional/?referralCode=E15D7FB64E417C547579
[103]: https://www.udemy.com/course/aws-cloudformation-basics?referralCode=93AD3B1530BC871093D6
[899]: https://www.udemy.com/user/n-kumar/
[900]: https://ko-fi.com/miztiik
[901]: https://ko-fi.com/Q5Q41QDGK
