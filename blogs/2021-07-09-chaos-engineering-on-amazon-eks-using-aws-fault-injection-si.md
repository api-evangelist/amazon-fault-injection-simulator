---
title: "Chaos engineering on Amazon EKS using AWS Fault Injection Simulator"
url: "https://aws.amazon.com/blogs/devops/chaos-engineering-on-amazon-eks-using-aws-fault-injection-simulator/"
date: "Fri, 09 Jul 2021 16:01:05 +0000"
author: "Omar Kahil"
feed_url: "https://aws.amazon.com/blogs/devops/tag/aws-fault-injection-simulator/feed/"
---
<p>In this post, we discuss how you can use <a href="https://aws.amazon.com/fis/">AWS Fault Injection Simulator</a> (AWS FIS), a fully managed fault injection service used for practicing chaos engineering. AWS FIS supports a range of AWS services, including <a href="https://aws.amazon.com/eks/">Amazon Elastic Kubernetes Service</a> (Amazon EKS), a managed service that helps you run Kubernetes on AWS without needing to install and operate your own Kubernetes control plane or worker nodes. In this post, we aim to show how you can simplify the process of setting up and running controlled fault injection experiments on Amazon EKS using pre-built templates as well as custom faults to find hidden weaknesses in your Amazon EKS workloads.</p> 
<h2><b>What is chaos engineering?</b></h2> 
<p>Chaos engineering is the process of stressing an application in testing or production environments by creating disruptive events, such as server outages or API throttling, observing how the system responds, and implementing improvements. Chaos engineering helps you create the real-world conditions needed to uncover the hidden issues and performance bottlenecks that are difficult to find in distributed systems. It starts with analyzing the steady-state behavior, building an experiment hypothesis (for example, stopping x number of instances will lead to x% more retries), running the experiment by injecting fault actions, monitoring rollback conditions, and addressing the weaknesses.</p> 
<p>AWS FIS lets you easily run fault injection experiments that are used in chaos engineering, making it easier to improve an application’s performance, observability, and resiliency.</p> 
<h2><b>Solution overview</b></h2> 
<div class="wp-caption alignnone" id="attachment_9950" style="width: 1034px;">
 <img alt="Figure 1: Solution Overview" class="wp-image-9950 size-large" height="486" src="https://d2908q01vomqb2.cloudfront.net/7719a1c782a1ba91c031a682a0a2f8658209adbf/2021/06/30/image1-1024x486.png" width="1024" />
 <p class="wp-caption-text" id="caption-attachment-9950">Figure 1: Solution Overview</p>
</div> 
<p>The following diagram illustrates our solution architecture.</p> 
<p>In this post, we demonstrate two different fault experiments targeting an Amazon EKS cluster. This post doesn’t go into details about the creation process of an Amazon EKS cluster; for more information, see <a href="https://docs.aws.amazon.com/eks/latest/userguide/getting-started-eksctl.html">Getting started with Amazon EKS – eksctl</a> and <a href="https://eksctl.io/">eksctl – The official CLI for Amazon EKS</a>.</p> 
<h2><b>Prerequisites</b></h2> 
<p>Before getting started, make sure you have the following prerequisites:</p> 
<ul> 
 <li>Access to an AWS account</li> 
 <li><a href="https://docs.aws.amazon.com/eks/latest/userguide/install-kubectl.html">kubectl</a> locally installed to interact with the Amazon EKS cluster</li> 
 <li>A running Amazon EKS cluster with <a href="https://docs.aws.amazon.com/eks/latest/userguide/cluster-autoscaler.html">Cluster Autoscaler</a> and <a href="https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/deploy-container-insights-EKS.html">Container Insights</a></li> 
 <li>Correct <a href="http://aws.amazon.com/iam">AWS Identity and Access Management</a> (IAM) permissions to work with AWS FIS (see <a href="https://docs.aws.amazon.com/fis/latest/userguide/getting-started-iam.html#getting-started-iam-users">Set up permissions for IAM users and roles</a>) and permissions for AWS FIS to run experiments on your behalf (see <a href="https://docs.aws.amazon.com/fis/latest/userguide/getting-started-iam.html#getting-started-iam-service-role">Set up the IAM role for the AWS FIS service</a>)</li> 
</ul> 
<p>We used the following configuration to create our cluster:</p> 
<pre><code class="lang-yaml">---
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: aws-fis-eks
  region: eu-west-1
  version: "1.19"

iam:
  withOIDC: true

managedNodeGroups:
- name: nodegroup
  desiredCapacity: 3
  instanceType: t3.small
  ssh:
    enableSsm: true
  tags:
    Environment: Dev</code></pre> 
<p>Our cluster was created with the following features:</p> 
<ul> 
 <li>Three <a href="http://aws.amazon.com/ec2">Amazon Elastic Compute Cloud</a> (Amazon EC2) t3.small instances spread across three different Availability Zones</li> 
 <li>Enabled <a href="https://docs.aws.amazon.com/eks/latest/userguide/enable-iam-roles-for-service-accounts.html">OIDC</a> provider</li> 
 <li>Enabled <a href="https://docs.aws.amazon.com/systems-manager/latest/userguide/ssm-agent.html">AWS Systems Manager Agent</a> on the instances (which we use later)</li> 
 <li>Tagged instances</li> 
</ul> 
<p>We have deployed a <a href="https://docs.aws.amazon.com/eks/latest/userguide/sample-deployment.html">simple Nginx deployment with three replicas</a>, each running on different instances for high availability.</p> 
<p>In this post, we perform the following experiments:</p> 
<ul> 
 <li><b>Terminate node group instances </b>–<b> </b>In the first experiment, we will use the <code>aws:eks:terminate-nodegroup-instance</code> AWS FIS action that runs the Amazon EC2 API action <a href="https://docs.aws.amazon.com/AWSEC2/latest/APIReference/API_TerminateInstances.html">TerminateInstances</a> on the target node group. When the experiment starts, AWS FIS begins to terminate nodes, and we should be able to verify that our cluster replaces the terminated nodes with new ones as per our desired capacity configuration for the cluster.</li> 
 <li><b>Delete application pods </b>–<b> </b>In the second experiment, we show how you can use AWS FIS to run custom faults against the cluster. Although AWS FIS plans to expand on supported faults for Amazon EKS in the future, in this example we demonstrate how you can run a custom fault injection, running <a href="https://docs.aws.amazon.com/eks/latest/userguide/install-kubectl.html">kubectl</a> commands to delete a random pod for our Kubernetes <a href="https://kubernetes.io/docs/concepts/workloads/controllers/deployment/">deployment</a>. Using a Kubernetes deployment is a good practice to define the desired state for the number of replicas you want to run for your application, and therefore ensures high availability in case one of the nodes or pods is stopped.</li> 
</ul> 
<h2><b>Experiment 1: Terminate node group instances</b></h2> 
<p>We start by creating an experiment to terminate Amazon EKS nodes.</p> 
<ol> 
 <li>On the AWS FIS console, choose <b>Create experiment template</b>.</li> 
</ol> 
<div class="wp-caption alignnone" id="attachment_9954" style="width: 1034px;">
 <img alt="Figure 2: AWS FIS Console" class="wp-image-9954 size-large" height="444" src="https://d2908q01vomqb2.cloudfront.net/7719a1c782a1ba91c031a682a0a2f8658209adbf/2021/06/30/Screenshot-2021-05-24-at-09.23.35-1024x444.png" width="1024" />
 <p class="wp-caption-text" id="caption-attachment-9954">Figure 2: AWS FIS Console</p>
</div> 
<p style="padding-left: 40px;">2. For <b>Description</b>, enter a description.</p> 
<p style="padding-left: 40px;">3. For <b>IAM role</b>, choose the IAM role you created.</p> 
<div class="wp-caption alignnone" id="attachment_9960" style="width: 819px;">
 <img alt="Figure 3: Create experiment template" class="wp-image-9960 size-full" height="339" src="https://d2908q01vomqb2.cloudfront.net/7719a1c782a1ba91c031a682a0a2f8658209adbf/2021/06/30/image2-1.png" width="809" />
 <p class="wp-caption-text" id="caption-attachment-9960">Figure 3: Create experiment template</p>
</div> 
<p style="padding-left: 40px;">&nbsp;&nbsp; 4. Choose <b>Add action</b>.</p> 
<p>For our action, we want <code>aws:eks:terminate-nodegroup-instances</code> to terminate worker nodes in our cluster.</p> 
<p style="padding-left: 40px;">&nbsp; 5. For <b>Name</b>, enter <code>TerminateWorkerNode</code>.</p> 
<p style="padding-left: 40px;">&nbsp; 6. For <b>Description</b>, enter <code>Terminate worker node</code>.</p> 
<p style="padding-left: 40px;">&nbsp; 7. For <b>Action type</b>, choose <b>aws:eks:terminate-nodegroup-instances</b>.</p> 
<p style="padding-left: 40px;">&nbsp; 8. For <b>Target</b>, choose <b>Nodegroups-Target-1</b>.</p> 
<p style="padding-left: 40px;">&nbsp; 9. For <b>instanceTerminationPercentage</b>, enter <code>40</code> (the percentage of instances that are terminated per node group).</p> 
<p style="padding-left: 40px;">&nbsp; 10. Choose <b>Save</b>.</p> 
<div class="wp-caption alignnone" id="attachment_9961" style="width: 812px;">
 <img alt="Figure 4: Select action type" class="wp-image-9961 size-full" height="729" src="https://d2908q01vomqb2.cloudfront.net/7719a1c782a1ba91c031a682a0a2f8658209adbf/2021/06/30/image3.png" width="802" />
 <p class="wp-caption-text" id="caption-attachment-9961">Figure 4: Select action type</p>
</div> 
<p>After you add the correct action, you can modify your target, which in this case is Amazon EKS node group instances.</p> 
<p style="padding-left: 40px;">11. Choose <b>Edit target</b>.</p> 
<p style="padding-left: 40px;">12. For <b>Resource type</b>, choose <b>aws:eks:nodegroup</b>.</p> 
<p style="padding-left: 40px;">13. For <b>Target method</b>, select <b>Resource IDs</b>.</p> 
<p style="padding-left: 40px;">14. For <b>Resource IDs</b>, enter your resource ID.</p> 
<p style="padding-left: 40px;">15. Choose <b>Save</b>.</p> 
<p>With <a href="https://docs.aws.amazon.com/fis/latest/userguide/targets.html#target-selection-mode">selection mode </a>in AWS FIS, you can select your Amazon EKS cluster node group.</p> 
<div class="wp-caption alignnone" id="attachment_9962" style="width: 834px;">
 <img alt="Figure 5: Specify target resource" class="wp-image-9962 size-full" height="540" src="https://d2908q01vomqb2.cloudfront.net/7719a1c782a1ba91c031a682a0a2f8658209adbf/2021/06/30/image4-2.png" width="824" />
 <p class="wp-caption-text" id="caption-attachment-9962">Figure 5: Specify target resource</p>
</div> 
<p>Finally, we add a <a href="https://docs.aws.amazon.com/fis/latest/userguide/stop-conditions.html">stop condition</a>. Even though this is optional, it’s highly recommended, because it makes sure we run experiments with the appropriate guardrails in place. The stop condition is a mechanism to stop an experiment if an <a href="http://aws.amazon.com/cloudwatch">Amazon CloudWatch</a> alarm reaches a threshold that you define. If a stop condition is triggered during an experiment, AWS FIS stops the experiment, and the experiment enters the <code>stopping</code> state.</p> 
<p>Because we have Container Insights configured for the cluster, we can monitor the number of nodes running in the cluster.</p> 
<p style="padding-left: 40px;">16. Through Container Insights, <a href="https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/AlarmThatSendsEmail.html">create a CloudWatch alarm</a> to stop our experiment if the number of nodes is less than two.</p> 
<p style="padding-left: 40px;">17. Add the alarm as a stop condition.</p> 
<p style="padding-left: 40px;">18. Choose <b>Create experiment template</b>.</p> 
<div class="wp-caption alignnone" id="attachment_9963" style="width: 1034px;">
 <img alt="Figure 6: Create experiment template" class="wp-image-9963 size-large" height="629" src="https://d2908q01vomqb2.cloudfront.net/7719a1c782a1ba91c031a682a0a2f8658209adbf/2021/06/30/Screenshot-2021-05-24-at-15.50.17-1024x629.png" width="1024" />
 <p class="wp-caption-text" id="caption-attachment-9963">Figure 6: Create experiment template</p>
</div> 
<div class="wp-caption alignnone" id="attachment_9964" style="width: 1034px;">
 <img alt="" class="wp-image-9964 size-large" height="158" src="https://d2908q01vomqb2.cloudfront.net/7719a1c782a1ba91c031a682a0a2f8658209adbf/2021/06/30/Screenshot-2021-06-14-at-21.02.19-1024x158.png" width="1024" />
 <p class="wp-caption-text" id="caption-attachment-9964">Figure 7: Check cluster nodes</p>
</div> 
<p>Before we run our first experiment, let’s check our Amazon EKS cluster nodes. In our case, we have three nodes up and running.</p> 
<p style="padding-left: 40px;">19. On the AWS FIS console, navigate to the details page for the experiment we created.</p> 
<p style="padding-left: 40px;">20. On the <b>Actions</b> menu, choose <b>Start</b>.</p> 
<div class="wp-caption alignnone" id="attachment_9965" style="width: 1034px;">
 <img alt="Figure 8: Start experiment" class="wp-image-9965 size-large" height="214" src="https://d2908q01vomqb2.cloudfront.net/7719a1c782a1ba91c031a682a0a2f8658209adbf/2021/06/30/image5-1-1024x214.png" width="1024" />
 <p class="wp-caption-text" id="caption-attachment-9965">Figure 8: Start experiment</p>
</div> 
<p>Before we run our experiment, AWS FIS will ask you to confirm if you want to start the experiment. This is another example of safeguards to make sure you’re ready to run an experiment against your resources.</p> 
<p style="padding-left: 40px;">21. Enter <code>start</code> in the field.</p> 
<p style="padding-left: 40px;">22. Choose <b>Start experiment</b>.</p> 
<div class="wp-caption alignnone" id="attachment_9966" style="width: 914px;">
 <img alt="Figure 9: Confirm to start experiment" class="wp-image-9966 size-full" height="538" src="https://d2908q01vomqb2.cloudfront.net/7719a1c782a1ba91c031a682a0a2f8658209adbf/2021/06/30/image6-1.png" width="904" />
 <p class="wp-caption-text" id="caption-attachment-9966">Figure 9: Confirm to start experiment</p>
</div> 
<p>After you start the experiment, you can see the experiment ID with its current state. You can also see the action the experiment is running.</p> 
<div class="wp-caption alignnone" id="attachment_9967" style="width: 1034px;">
 <img alt="Figure 10: Check experiment state" class="wp-image-9967 size-large" height="418" src="https://d2908q01vomqb2.cloudfront.net/7719a1c782a1ba91c031a682a0a2f8658209adbf/2021/06/30/image7-1-1024x418.png" width="1024" />
 <p class="wp-caption-text" id="caption-attachment-9967">Figure 10: Check experiment state</p>
</div> 
<p>Next, we can check the status of our cluster worker nodes. The process of adding a new node to the cluster takes a few minutes, but after a while we can see that Amazon EKS has launched new instances to replace the terminated ones.</p> 
<p>The number of terminated instances should reflect the percentage that we provided as part of our action configuration. Because our experiment is complete, we can verify our hypothesis—our cluster eventually reached a steady state with a number of nodes equal to the desired capacity within a few minutes.</p> 
<div class="wp-caption alignnone" id="attachment_9968" style="width: 1034px;">
 <img alt="Figure 11: Check new worker node" class="wp-image-9968 size-large" height="151" src="https://d2908q01vomqb2.cloudfront.net/7719a1c782a1ba91c031a682a0a2f8658209adbf/2021/06/30/Screenshot-2021-06-14-at-21.08.59-1024x151.png" width="1024" />
 <p class="wp-caption-text" id="caption-attachment-9968">Figure 11: Check new worker node</p>
</div> 
<h2><b>Experiment 2: Delete application pods</b></h2> 
<p>Now, let’s create a custom fault injection, targeting a specific containerized application (pod) running on our Amazon EKS cluster.</p> 
<p>As a prerequisite for this experiment, you need to update your Amazon EKS cluster configmap, adding the IAM role that is attached to your worker nodes. The reason for adding this role to the configmap is because the experiment uses kubectl, the Kubernetes command-line tool that allows us to run commands against our Kubernetes cluster. For instructions, see <a href="https://docs.aws.amazon.com/eks/latest/userguide/add-user-role.html">Managing users or IAM roles for your cluster</a>.</p> 
<ol> 
 <li>On the Systems Manager console, choose <b>Documents</b>.</li> 
 <li>On the <b>Create document</b> menu, choose <b>Command or Session</b>.</li> 
 <li> 
  <ol> 
   <li></li> 
  </ol> </li> 
</ol> 
<div class="wp-caption alignnone" id="attachment_9969" style="width: 1034px;">
 <img alt="Figure 12: Create AWS Systems Manager Document" class="wp-image-9969 size-large" height="178" src="https://d2908q01vomqb2.cloudfront.net/7719a1c782a1ba91c031a682a0a2f8658209adbf/2021/06/30/image8-1-1024x178.png" width="1024" />
 <p class="wp-caption-text" id="caption-attachment-9969">Figure 12: Create AWS Systems Manager Document</p>
</div> 
<p style="padding-left: 40px;">3. For <b>Name</b>, enter a name (for example, <code>Delete-Pods</code>).</p> 
<p style="padding-left: 40px;">4. In the <b>Content</b> section, enter the following code:</p> 
<pre><code class="lang-yaml">---
description: |
  ### Document name - Delete Pod

  ## What does this document do?
  Delete Pod in a specific namespace via kubectl

  ## Input Parameters
  * Cluster: (Required)
  * Namespace: (Required)
  * InstallDependencies: If set to True, Systems Manager installs the required dependencies on the target instances. (default True)

  ## Output Parameters
  None.

schemaVersion: '2.2'
parameters:
  Cluster:
    type: String
    description: '(Required) Specify the cluster name'
  Namespace:
    type: String
    description: '(Required) Specify the target Namespace'
  InstallDependencies:
    type: String
    description: 'If set to True, Systems Manager installs the required dependencies on the target instances (default: True)'
    default: 'True'
    allowedValues:
      - 'True'
      - 'False'
mainSteps:
  - action: aws:runShellScript
    name: InstallDependencies
    precondition:
      StringEquals:
        - platformType
        - Linux
    description: |
      ## Parameter: InstallDependencies
      If set to True, this step installs the required dependecy via operating system's repository.
    inputs:
      runCommand:
        - |
          #!/bin/bash
          if [[ "{{ InstallDependencies }}" == True ]] ; then
            if [[ "$( which kubectl 2&gt;/dev/null )" ]] ; then echo Dependency is already installed. ; exit ; fi
            echo "Installing required dependencies"
            sudo mkdir -p $HOME/bin &amp;&amp; cd $HOME/bin
            sudo curl -o kubectl https://amazon-eks.s3.us-west-2.amazonaws.com/1.20.4/2021-04-12/bin/linux/amd64/kubectl
            sudo chmod +x ./kubectl
            export PATH=$PATH:$HOME/bin
          fi
  - action: aws:runShellScript
    name: ExecuteKubectlDeletePod
    precondition:
      StringEquals:
        - platformType
        - Linux
    description: |
      ## Parameters: Namespace, Cluster, Namespace
      This step will terminate the random first pod based on namespace provided
    inputs:
      maxAttempts: 1
      runCommand:
        - |
          if [ -z "{{ Cluster }}" ] ; then echo Cluster not specified &amp;&amp; exit; fi
          if [ -z "{{ Namespace }}" ] ; then echo Namespace not specified &amp;&amp; exit; fi
          pgrep kubectl &amp;&amp; echo Another kubectl command is already running, exiting... &amp;&amp; exit
          EC2_REGION=$(curl -s http://169.254.169.254/latest/dynamic/instance-identity/document|grep region | awk -F\" '{print $4}')
          aws eks --region $EC2_REGION update-kubeconfig --name {{ Cluster }} --kubeconfig /home/ssm-user/.kube/config
          echo Running kubectl command...
          TARGET_POD=$(kubectl --kubeconfig /home/ssm-user/.kube/config get pods -n {{ Namespace }} -o jsonpath={.items[0].metadata.name})
          echo "TARGET_POD: $TARGET_POD"
          kubectl --kubeconfig /home/ssm-user/.kube/config delete pod $TARGET_POD -n {{ Namespace }} --grace-period=0 --force
          echo Finished kubectl delete pod command.</code></pre> 
<pre></pre> 
<div class="wp-caption alignnone" id="attachment_9970" style="width: 1034px;">
 <img alt="Figure 13: Add Document details" class="wp-image-9970 size-large" height="543" src="https://d2908q01vomqb2.cloudfront.net/7719a1c782a1ba91c031a682a0a2f8658209adbf/2021/06/30/image9-1-1024x543.png" width="1024" />
 <p class="wp-caption-text" id="caption-attachment-9970">Figure 13: Add Document details</p>
</div> 
<p>For this post, we create a Systems Manager <a href="https://docs.aws.amazon.com/systems-manager/latest/userguide/sysman-ssm-docs.html">command document</a> that does the following:</p> 
<ul> 
 <li>Installs kubectl on the target Amazon EKS cluster instances</li> 
 <li>Uses two required parameters—the Amazon EKS cluster name and namespace where your application pods are running</li> 
 <li>Runs kubectl delete, deleting one of our application pods from a specific namespace</li> 
</ul> 
<p style="padding-left: 40px;">5. Choose <b>Create document.</b></p> 
<p style="padding-left: 40px;">6. Create a new experiment template on the AWS FIS console.</p> 
<p style="padding-left: 40px;">7. For <b>Name</b>, enter <code>DeletePod</code>.</p> 
<p style="padding-left: 40px;">8. For <b>Action type</b>, choose <b>aws:ssm:send-command.</b></p> 
<p>This runs the Systems Manager API action <code>SendCommand</code> to our target EC2 instances.</p> 
<p>After choosing this action, we need to provide the ARN for the document we created earlier, and provide the appropriate values for the cluster and namespace. In our example, we named the document <code>Delete-Pods</code>, our cluster name is<b> </b><code>aws-fis-eks</code>, and our namespace is <code>nginx</code>.</p> 
<p style="padding-left: 40px;">9. For <b>documentARN</b>, enter <code>arn:aws:ssm:</code><i>&lt;region&gt;</i><code>:</code><i>&lt;accountId&gt;</i><code>:document/Delete-Pods</code>.</p> 
<p style="padding-left: 40px;">10. For <b>documentParameters</b>, enter <code>{"Cluster":"aws-fis-eks", "Namespace":"nginx", "InstallDependencies":"True"}</code>.</p> 
<p style="padding-left: 40px;">11. Choose <b>Save</b>.</p> 
<div class="wp-caption alignnone" id="attachment_9971" style="width: 1034px;">
 <img alt="Figure 14: Select Action type" class="wp-image-9971 size-large" height="884" src="https://d2908q01vomqb2.cloudfront.net/7719a1c782a1ba91c031a682a0a2f8658209adbf/2021/06/30/image10-1-1024x884.png" width="1024" />
 <p class="wp-caption-text" id="caption-attachment-9971">Figure 14: Select Action type</p>
</div> 
<p style="padding-left: 40px;">12. For our targets, we can either target our resources by resource IDs or resource tags. For this example we target one of our node instances by resource ID.</p> 
<div class="wp-caption alignnone" id="attachment_9972" style="width: 1034px;">
 <img alt="Figure 15: Specify target resource" class="wp-image-9972 size-large" height="651" src="https://d2908q01vomqb2.cloudfront.net/7719a1c782a1ba91c031a682a0a2f8658209adbf/2021/06/30/image11-1-1024x651.png" width="1024" />
 <p class="wp-caption-text" id="caption-attachment-9972">Figure 15: Specify target resource</p>
</div> 
<p style="padding-left: 40px;">13. After you create the template successfully, start the experiment.</p> 
<p>When the experiment is complete, check your application pods. In our case, AWS FIS stopped one of our pod replicas and because we use a Kubernetes deployment, as we discussed before, a new pod replica was created.</p> 
<div class="wp-caption alignnone" id="attachment_9973" style="width: 1034px;">
 <img alt="Figure 16: Check Deployment pods" class="wp-image-9973 size-large" height="411" src="https://d2908q01vomqb2.cloudfront.net/7719a1c782a1ba91c031a682a0a2f8658209adbf/2021/06/30/Screenshot-2021-05-21-at-15.45.54-1024x411.png" width="1024" />
 <p class="wp-caption-text" id="caption-attachment-9973">Figure 16: Check Deployment pods</p>
</div> 
<h2><b>Clean up</b></h2> 
<p style="text-align: left;">To avoid incurring future charges, follow the steps below to remove all resources that was created following along with this post.</p> 
<ol> 
 <li style="text-align: left;">From the AWS FIS console, delete the following experiments, TerminateWorkerNodes &amp; DeletePod.</li> 
 <li style="text-align: left;">From the AWS EKS console, delete the test cluster created following this post, aws-fis-eks.</li> 
 <li style="text-align: left;">From the AWS Identity and Access Management (IAM) console, delete the IAM role AWSFISRole.</li> 
 <li style="text-align: left;">From the Amazon CloudWatch console, delete the CloudWatch alarm CheckEKSNodes.</li> 
 <li style="text-align: left;">From the AWS Systems Manager console, delete the Owned by me document Delete-Pods.</li> 
</ol> 
<h2><b>Conclusion</b></h2> 
<p>In this post, we showed two ways you can run fault injection experiments on Amazon EKS using AWS FIS. First, we used a native action supported by AWS FIS to terminate instances from our Amazon EKS cluster. Then, we extended AWS FIS to inject custom faults on our containerized applications running on Amazon EKS.</p> 
<p>For more information about AWS FIS, check out the AWS re:Invent 2020 session <a href="https://www.youtube.com/watch?v=yoNeMLj3CHc">AWS Fault Injection Simulator: Fully managed chaos engineering service</a>. If you want to know more about chaos engineering, check out the AWS re:Invent session <a href="https://www.youtube.com/watch?v=OlobVYPkxgg&amp;ab_channel=AWSEvents">Testing resiliency using chaos engineering</a> and <a href="https://medium.com/the-cloud-architect/the-chaos-engineering-collection-5e188d6a90e2">The Chaos Engineering Collection</a>. Finally, check out the following <a href="https://github.com/aws-samples/aws-fault-injection-simulator-samples">GitHub repo</a> for additional example experiments, and how you can work with AWS FIS using the <a href="https://docs.aws.amazon.com/cdk/latest/guide/home.html">AWS Cloud Development Kit</a> (AWS CDK).</p> 
<p>&nbsp;</p> 
<h3>About the authors</h3> 
<p><img alt="" class="alignleft wp-image-8385 size-thumbnail" height="150" src="https://d2908q01vomqb2.cloudfront.net/7719a1c782a1ba91c031a682a0a2f8658209adbf/2021/03/17/Screenshot-2021-03-17-at-20.26.06-150x150.png" width="150" /></p> 
<p>&nbsp;</p> 
<p><em>Omar is a Professional Services consultant who helps customers adopt DevOps culture and best practices. He also works to simplify the adoption of AWS services by automating and implementing complex solutions.</em></p> 
<p>&nbsp;</p> 
<p>&nbsp;</p> 
<p>&nbsp;</p> 
<p><img alt="" class="size-full wp-image-10035 alignleft" height="150" src="https://d2908q01vomqb2.cloudfront.net/7719a1c782a1ba91c031a682a0a2f8658209adbf/2021/07/05/myphoto_150.png" width="150" /></p> 
<p>&nbsp;</p> 
<p><em>Daniel Arenhage is a Solutions Architect at Amazon Web Services based in Gothenburg, Sweden.</em></p> 
<p>&nbsp;</p>
