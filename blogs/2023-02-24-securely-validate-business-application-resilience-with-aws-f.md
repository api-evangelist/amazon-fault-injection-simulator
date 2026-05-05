---
title: "Securely validate business application resilience with AWS FIS and IAM"
url: "https://aws.amazon.com/blogs/devops/securely-validate-business-application-resilience-with-aws-fis-and-iam/"
date: "Fri, 24 Feb 2023 20:15:35 +0000"
author: "Dr. Rudolf Potucek"
feed_url: "https://aws.amazon.com/blogs/devops/tag/aws-fault-injection-simulator/feed/"
---
<p>To avoid high costs of downtime, mission critical applications in the cloud need to achieve resilience against degradation of cloud provider APIs and services.</p> 
<p>In 2021, AWS launched&nbsp;<a href="https://docs.aws.amazon.com/fis/latest/userguide/what-is.html">AWS Fault Injection Simulator</a><span style="color: #16191f;"> (FIS), a fully managed service to perform fault injection experiments on workloads in AWS to&nbsp;improve their </span><a href="https://docs.aws.amazon.com/wellarchitected/latest/reliability-pillar/welcome.html"><span style="color: #16191f;">reliability</span></a><span style="color: #16191f;">&nbsp;and resilience.</span><span style="color: #16191f;">&nbsp;</span><span style="color: #16191f;">At the time of writing,&nbsp;FIS allows to simulate degradation of </span><a href="https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/concepts.html">Amazon Elastic Compute Cloud</a><span style="color: #16191f;"> (EC2) APIs using </span><a href="https://docs.aws.amazon.com/fis/latest/userguide/fis-actions-reference.html"><span style="color: #16191f;">API fault injection actions</span></a><span style="color: #16191f;">&nbsp;and thus explore the resilience of workflows where EC2 APIs act as a fault boundary.&nbsp;</span></p> 
<p>In this post we show you how to explore additional fault boundaries in your applications by selectively denying access to any AWS API. This technique is particularly useful for fully managed, “black box” services like <a href="https://docs.aws.amazon.com/AmazonS3/latest/userguide/Welcome.html">Amazon Simple Storage Service</a> (S3) or <a href="https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/welcome.html">Amazon Simple Queue Service</a> (SQS) where a failure of read or write operations is sufficient to simulate problems in the service. This technique is also&nbsp;<span style="color: #16191f;">useful for injecting failures in serverless applications without needing to modify code. </span><span style="color: #16191f;">While similar results could be achieved with network disruption or modifying code with feature flags, this approach provides a fine granular degradation of an AWS API without the need to re-deploy and re-validate code.</span></p> 
<h2 id="temp:C:bBc79a3c36e0c4b42d9850c1082f">Overview</h2> 
<p><span style="color: #16191f;">We will explore a common application pattern: user uploads a file, S3 triggers an </span><a href="https://docs.aws.amazon.com/lambda/latest/dg/welcome.html">AWS Lambda</a><span style="color: #16191f;"> function, Lambda transforms the file to a new location and deletes the original:</span></p> 
<div class=""> 
 <div class="wp-caption aligncenter" id="attachment_14643" style="width: 710px;">
  <img alt="S3 upload and transform logical workflow: User uploads file to S3, upload triggers AWS Lambda execution, Lambda writes transformed file to a new bucket and deletes original. Workflow can be disrupted at file deletion." class="wp-image-14643" height="223" src="https://d2908q01vomqb2.cloudfront.net/7719a1c782a1ba91c031a682a0a2f8658209adbf/2023/02/22/fis-iam-api-blog-diagram-0.png" width="700" />
  <p class="wp-caption-text" id="caption-attachment-14643">Figure 1. S3 upload and transform logical workflow: User uploads file to S3, upload triggers AWS Lambda execution, Lambda writes transformed file to a new bucket and deletes original. Workflow can be disrupted at file deletion.</p>
 </div> 
</div> 
<p>We will simulate the user upload with an <a href="https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-what-is.html">Amazon EventBridge</a> <a href="https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-create-rule-schedule.html#eb-rate-expressions">rate expression</a>&nbsp;triggering an AWS Lambda function which creates a file in S3:</p> 
<div class=""> 
 <div class="wp-caption aligncenter" id="attachment_14644" style="width: 710px;">
  <img alt="S3 upload and transform implemented demo workflow: Amazon EventBridge triggers a creator Lambda function, Lambda function creates a file in S3, file creation triggers AWS Lambda execution on transformer function, Lambda writes transformed file to a new bucket and deletes original. Workflow can be disrupted at file deletion." class="wp-image-14644" height="291" src="https://d2908q01vomqb2.cloudfront.net/7719a1c782a1ba91c031a682a0a2f8658209adbf/2023/02/22/fis-iam-api-blog-diagram-1.png" width="700" />
  <p class="wp-caption-text" id="caption-attachment-14644">Figure 2. S3 upload and transform implemented demo workflow: Amazon EventBridge triggers a creator Lambda function, Lambda function creates a file in S3, file creation triggers AWS Lambda execution on transformer function, Lambda writes transformed file to a new bucket and deletes original. Workflow can be disrupted at file deletion.</p>
 </div> 
</div> 
<p>Using this architecture we can explore the effect of S3 API degradation during file creation and deletion. As shown, the API call to delete a file from S3 is an application fault boundary. The failure could occur, with identical effect, because of S3 degradation or because the <a href="https://docs.aws.amazon.com/IAM/latest/UserGuide/introduction.html">AWS IAM</a> role of the Lambda function denies access to the API.</p> 
<p><span style="color: #16191f;">To inject failures we use&nbsp;</span><a href="https://docs.aws.amazon.com/systems-manager/latest/userguide/what-is-systems-manager.html">AWS Systems Manager</a> (AWS SSM) <a href="https://docs.aws.amazon.com/systems-manager/latest/userguide/systems-manager-automation.html">automation documents</a><span style="color: #16191f;"> to attach and detach IAM policies at the API fault boundary and FIS to orchestrate&nbsp;</span><span style="color: #16191f;">the workflow</span><span style="color: #16191f;">.</span></p> 
<p>Each Lambda function has an IAM execution role that allows S3 write and delete access, respectively. If the processor Lambda fails, the S3 file will remain in the bucket, indicating a failure. Similarly, if the IAM execution role for the processor function is denied the ability to delete a file after processing, that file will remain in the S3 bucket.</p> 
<h2 id="temp:C:bBc7c2de5f93ea14251a03d80262">Prerequisites</h2> 
<p>Following this blog posts will incur some costs for AWS services. To explore this test application you will need an <a href="https://aws.amazon.com/premiumsupport/knowledge-center/create-and-activate-aws-account/">AWS account</a>. We will also assume that you are using <a href="https://console.aws.amazon.com/cloudshell/home">AWS CloudShell</a> or have the AWS CLI <a href="https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html">installed</a>&nbsp;and have <a href="https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-quickstart.html">configured</a> a profile with administrator permissions.&nbsp;With that in place you can create the demo application in your AWS account by downloading this <a href="https://github.com/aws-samples/fis-api-failure-injection-using-iam/blob/main/template.yaml">template</a>&nbsp;and deploying an <a href="https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/Welcome.html">AWS CloudFormation</a> stack:</p> 
<pre><code class="lang-bash">git clone https://github.com/aws-samples/fis-api-failure-injection-using-iam.git
cd fis-api-failure-injection-using-iam
aws cloudformation deploy --stack-name test-fis-api-faults --template-file template.yaml --capabilities CAPABILITY_NAMED_IAM</code></pre> 
<h2 id="temp:C:bBc81024771106d498383d803156">Fault injection using IAM</h2> 
<p>Once the stack has been created, navigate to the <a href="https://console.aws.amazon.com/cloudwatch/home?#logsV2:log-groups$3FlogGroupNameFilter$3D$252Faws$252Flambda$252Ftest-fis-api-faults">Amazon CloudWatch Logs console</a> and filter for&nbsp;<code>/aws/lambda/test-fis-api-faults</code>. Under the <code>EventBridgeTimerHandler</code>&nbsp;log group you should find log events once a minute writing a timestamped file to an S3 bucket named <code>fis-api-failure-ACCOUNT_ID</code>. Under the <code>S3TriggerHandler</code>&nbsp;log group you should find matching deletion events for those files.</p> 
<p>Once you have confirmed object creation/deletion, let’s take away the permission of the S3 trigger handler lambda to delete files. To do this you will attach the <code>FISAPI-DenyS3DeleteObject</code>&nbsp;&nbsp;policy that was created with the template:</p> 
<pre><code class="lang-bash">ROLE_NAME=FISAPI-TARGET-S3TriggerHandlerRole
ROLE_ARN=$( aws iam list-roles --query "Roles[?RoleName=='${ROLE_NAME}'].Arn" --output text )
echo Target Role ARN: $ROLE_ARN

POLICY_NAME=FISAPI-DenyS3DeleteObject
POLICY_ARN=$( aws iam list-policies --query "Policies[?PolicyName=='${POLICY_NAME}'].Arn" --output text )
echo Impact Policy ARN: $POLICY_ARN

aws iam attach-role-policy \
  --role-name ${ROLE_NAME}\
  --policy-arn ${POLICY_ARN}</code></pre> 
<p id="temp:C:bBc9ab4f2ed89f842b6b01836a4c">With the deny policy in place you should now see object deletion fail and objects should start showing up in the S3 bucket. Navigate to the S3 console and find the bucket starting with <code>fis-api-failure</code>. You should see a new object appearing in this bucket once a minute:</p> 
<div class="wp-caption aligncenter" id="attachment_14654" style="width: 710px;">
 <img alt="S3 bucket listing showing files not being deleted because IAM permissions DENY file deletion during FIS experiment. " class="wp-image-14654" height="201" src="https://d2908q01vomqb2.cloudfront.net/7719a1c782a1ba91c031a682a0a2f8658209adbf/2023/02/22/fis-iam-api-s3-files.png" width="700" />
 <p class="wp-caption-text" id="caption-attachment-14654">Figure 3. S3 bucket listing showing files not being deleted because IAM permissions DENY file deletion during FIS experiment.</p>
</div> 
<p>If you would like to graph the results you can navigate to <a href="https://console.aws.amazon.com/cloudwatch/home?#logsV2:logs-insights">AWS CloudWatch</a>, select “Logs Insights“, select the log group starting with <span style="color: #16191f;"><span style="background-color: #f1faff;"><code>/aws/lambda/test-fis-api-faults-S3CountObjectsHandler</code></span></span>, and run this query:</p> 
<pre><code class="lang-cloudwatchlogsinsights">fields @timestamp, @message
| filter NumObjects &gt;= 0
| sort @timestamp desc
| stats max(NumObjects) by bin(1m)
| limit 20</code></pre> 
<p>This will show the number of files in the S3 bucket over time:</p> 
<div class="wp-caption aligncenter" id="attachment_14648" style="width: 710px;">
 <img alt="AWS CloudWatch Logs Insights graph showing the increase in the number of retained files in S3 bucket over time, demonstrating the effect of the introduced failure." class="wp-image-14648" height="588" src="https://d2908q01vomqb2.cloudfront.net/7719a1c782a1ba91c031a682a0a2f8658209adbf/2023/02/22/fis-iam-api-s3-log-insights.png" width="700" />
 <p class="wp-caption-text" id="caption-attachment-14648">Figure 4. AWS CloudWatch Logs Insights graph showing the increase in the number of retained files in S3 bucket over time, demonstrating the effect of the introduced failure.</p>
</div> 
<p>You can now detach the policy:</p> 
<pre><code class="lang-bash">ROLE_NAME=FISAPI-TARGET-S3TriggerHandlerRole
ROLE_ARN=$( aws iam list-roles --query "Roles[?RoleName=='${ROLE_NAME}'].Arn" --output text )
echo Target Role ARN: $ROLE_ARN

POLICY_NAME=FISAPI-DenyS3DeleteObject
POLICY_ARN=$( aws iam list-policies --query "Policies[?PolicyName=='${POLICY_NAME}'].Arn" --output text )
echo Impact Policy ARN: $POLICY_ARN

aws iam detach-role-policy \
  --role-name ${ROLE_NAME}\
  --policy-arn ${POLICY_ARN}</code></pre> 
<p>We see that newly written files will once again be deleted but the un-processed files will remain in the S3 bucket. From the fault injection we learned that our system does not tolerate request failures when deleting files from S3.&nbsp;To address this, we should add a dead letter queue or some other retry mechanism.</p> 
<p>Note: if the Lambda function does not return a success state on invocation, EventBridge will <a href="https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-rule-dlq.html">retry</a>. In our Lambda functions we are cost conscious and explicitly capture the failure states to avoid&nbsp;excessive retries.</p> 
<h2 id="temp:C:bBc65ff2e4c591244e2aa2e46974">Fault injection using SSM</h2> 
<p>To use this approach from FIS and to always remove the policy at the end of the experiment, we first create an SSM document to automate adding a policy to a role. To inspect this document, open the SSM console, navigate to the “Documents” section, find the <a href="https://console.aws.amazon.com/systems-manager/documents/FISAPI-IamAttachDetach/content">FISAPI-IamAttachDetach</a> document under “Owned by me”, and examine the “Content” tab (make sure to select the correct region). This document takes the name of the Role you want to impact and the Policy you want to attach as parameters. It also requires an IAM execution role that grants it the power to list, attach, and detach specific policies to specific roles.</p> 
<p>Let’s run the SSM automation document from the console by selecting “Execute Automation”. Determine the ARN of the <code>FISAPI-SSM-Automation-Role</code>&nbsp;from CloudFormation or by running:</p> 
<pre><code class="lang-bash">POLICY_NAME=FISAPI-DenyS3DeleteObject
POLICY_ARN=$( aws iam list-policies --query "Policies[?PolicyName=='${POLICY_NAME}'].Arn" --output text )
echo Impact Policy ARN: $POLICY_ARN</code></pre> 
<p id="temp:C:bBc06d64af1e59f4fc291ef8299a">Use <code>FISAPI-SSM-Automation-Role</code>, a duration of 2 minutes expressed in ISO8601 format as <code>PT2M</code>, the ARN of the deny policy, and the name of the target role <code>FISAPI-TARGET-S3TriggerHandlerRole</code>:</p> 
<div> 
 <div class="wp-caption aligncenter" id="attachment_14649" style="width: 710px;">
  <img alt="Image of parameter input field reflecting the instructions in blog text." class="wp-image-14649" height="165" src="https://d2908q01vomqb2.cloudfront.net/7719a1c782a1ba91c031a682a0a2f8658209adbf/2023/02/23/fis-iam-api-ssm-start-automation.png" width="700" />
  <p class="wp-caption-text" id="caption-attachment-14649">Figure 5. Image of parameter input field reflecting the instructions in blog text.</p>
 </div> 
</div> 
<p>Alternatively&nbsp;execute this from a shell:</p> 
<pre><code class="lang-bash">ASSUME_ROLE_NAME=FISAPI-SSM-Automation-Role
ASSUME_ROLE_ARN=$( aws iam list-roles --query "Roles[?RoleName=='${ASSUME_ROLE_NAME}'].Arn" --output text )
echo Assume Role ARN: $ASSUME_ROLE_ARN

ROLE_NAME=FISAPI-TARGET-S3TriggerHandlerRole
ROLE_ARN=$( aws iam list-roles --query "Roles[?RoleName=='${ROLE_NAME}'].Arn" --output text )
echo Target Role ARN: $ROLE_ARN

POLICY_NAME=FISAPI-DenyS3DeleteObject
POLICY_ARN=$( aws iam list-policies --query "Policies[?PolicyName=='${POLICY_NAME}'].Arn" --output text )
echo Impact Policy ARN: $POLICY_ARN

aws ssm start-automation-execution \
  --document-name FISAPI-IamAttachDetach \
  --parameters "{
      \"AutomationAssumeRole\": [ \"${ASSUME_ROLE_ARN}\" ],
      \"Duration\": [ \"PT2M\" ],
      \"TargetResourceDenyPolicyArn\": [\"${POLICY_ARN}\" ],
      \"TargetApplicationRoleName\": [ \"${ROLE_NAME}\" ]
    }"
</code></pre> 
<p>Wait two minutes&nbsp;and then examine the content of the S3&nbsp;bucket starting with <code>fis-api-failure</code>&nbsp;again. You should now see two additional files in the bucket, showing that the policy was attached for 2 minutes during which files could not be deleted, and confirming that our application is not resilient to S3 API degradation.</p> 
<h2 id="temp:C:bBcd8af2056ba904763927765d5f">Permissions for injecting failures with SSM</h2> 
<p>Fault injection with SSM is controlled by IAM, which is why&nbsp;you had to specify the <code>FISAPI-SSM-Automation-Role</code>:</p> 
<div class="wp-caption aligncenter" id="attachment_14645" style="width: 710px;">
 <img alt="Visual representation of IAM permission used for fault injections with SSM. It shows the SSM execution role permitting access to use SSM automation documents as well as modify IAM roles and policies via the SSM document. It also shows the SSM user needing to have a pass-role permission to grant the SSM execution role to the SSM service." class="wp-image-14645" height="307" src="https://d2908q01vomqb2.cloudfront.net/7719a1c782a1ba91c031a682a0a2f8658209adbf/2023/02/22/fis-iam-api-blog-diagram-2.png" width="700" />
 <p class="wp-caption-text" id="caption-attachment-14645">Figure 6. Visual representation of IAM permission used for fault injections with SSM.</p>
</div> 
<p>This role needs to contain an assume role policy statement for SSM to allow assuming the role:</p> 
<pre><code class="lang-yaml">      AssumeRolePolicyDocument:
        Statement:
          - Action:
             - 'sts:AssumeRole'
            Effect: Allow
            Principal:
              Service:
                - "ssm.amazonaws.com"</code></pre> 
<p>The role also needs to contain permissions to describe roles and their attached policies with an optional constraint on which roles and policies are visible:</p> 
<pre><code class="lang-yaml">          - Sid: GetRoleAndPolicyDetails
            Effect: Allow
            Action:
              - 'iam:GetRole'
              - 'iam:GetPolicy'
              - 'iam:ListAttachedRolePolicies'
            Resource:
              # Roles
              - !GetAtt EventBridgeTimerHandlerRole.Arn
              - !GetAtt S3TriggerHandlerRole.Arn
              # Policies
              - !Ref AwsFisApiPolicyDenyS3DeleteObject</code></pre> 
<p>Finally the SSM role needs to allow attaching and detaching a policy document. This requires</p> 
<div class=""> 
 <ol id="temp:C:bBc6594e114ecd64430a00a27b87"> 
  <li class="" id="temp:C:bBc87468a635c3e403489e05fe0a" value="1">an ALLOW statement</li> 
  <li class="" id="temp:C:bBca4b62c3b35c84a0dad0a57a3c">a constraint on the policies that can be attached</li> 
  <li class="" id="temp:C:bBc16c7368f935b4a3ab02ae9c3b">a constraint on the roles that can be attached to</li> 
 </ol> 
</div> 
<p>In the role we collapse the first two requirements into an ALLOW statement with a condition constraint for the Policy ARN. We then express the third requirement in a DENY statement that will limit the <code>'*'</code>&nbsp;resource to only the explicit&nbsp;role ARNs we want to modify:</p> 
<pre><code class="lang-yaml">          - Sid: AllowOnlyTargetResourcePolicies
            Effect: Allow
            Action:  
              - 'iam:DetachRolePolicy'
              - 'iam:AttachRolePolicy'
            Resource: '*'
            Condition:
              ArnEquals:
                'iam:PolicyARN':
                  # Policies that can be attached
                  - !Ref AwsFisApiPolicyDenyS3DeleteObject
          - Sid: DenyAttachDetachAllRolesExceptApplicationRole
            Effect: Deny
            Action: 
              - 'iam:DetachRolePolicy'
              - 'iam:AttachRolePolicy'
            NotResource: 
              # Roles that can be attached to
              - !GetAtt EventBridgeTimerHandlerRole.Arn
              - !GetAtt S3TriggerHandlerRole.Arn</code></pre> 
<p>We will discuss security considerations in more detail at the end of this post.</p> 
<h2 id="temp:C:bBcef488f54dd2e4f81beefc1349">Fault injection using FIS</h2> 
<p>With the SSM document in place you can now create an FIS template that calls the SSM document. Navigate to the <a href="https://console.aws.amazon.com/fis/home?#ExperimentTemplates:v=3&amp;tag:Name=FISAPI-DENY-S3PutObject">FIS console</a> and filter for <code>FISAPI-DENY-S3PutObject</code>. You should see that the experiment template passes the same parameters that you previously used with SSM:</p> 
<div class="wp-caption aligncenter" id="attachment_14647" style="width: 710px;">
 <img alt="Image of FIS experiment template action summary. This shows the SSM document ARN to be used for fault injection and the JSON parameters passed to the SSM document specifying the IAM Role to modify and the IAM Policy to use. " class="wp-image-14647" height="485" src="https://d2908q01vomqb2.cloudfront.net/7719a1c782a1ba91c031a682a0a2f8658209adbf/2023/02/23/fis-iam-api-fis-explore-template.png" width="700" />
 <p class="wp-caption-text" id="caption-attachment-14647">Figure 7. Image of FIS experiment template action summary. This shows the SSM document ARN to be used for fault injection and the JSON parameters passed to the SSM document specifying the IAM Role to modify and the IAM Policy to use.</p>
</div> 
<p>You can now run the FIS experiment and after a couple minutes once again see new files in the S3 bucket.</p> 
<h2 id="temp:C:bBc4593db2afa524b7ab376609be">Permissions for injecting failures with FIS and SSM</h2> 
<p>Fault injection with FIS is controlled by IAM, which is why&nbsp;you had to specify&nbsp;the <code>FISAPI-FIS-Injection-EperimentRole</code>:</p> 
<div> 
 <div class="wp-caption aligncenter" id="attachment_14646" style="width: 710px;">
  <img alt="Visual representation of IAM permission used for fault injections with FIS and SSM. It shows the SSM execution role permitting access to use SSM automation documents as well as modify IAM roles and policies via the SSM document. It also shows the FIS execution role permitting access to use FIS templates, as well as the pass-role permission to grant the SSM execution role to the SSM service. Finally it shows the FIS user needing to have a pass-role permission to grant the FIS execution role to the FIS service." class="wp-image-14646" height="307" src="https://d2908q01vomqb2.cloudfront.net/7719a1c782a1ba91c031a682a0a2f8658209adbf/2023/02/23/fis-iam-api-blog-diagram-3.png" width="700" />
  <p class="wp-caption-text" id="caption-attachment-14646">Figure 8. Visual representation of IAM permission used for fault injections with FIS and SSM. It shows the SSM execution role permitting access to use SSM automation documents as well as modify IAM roles and policies via the SSM document. It also shows the FIS execution role permitting access to use FIS templates, as well as the pass-role permission to grant the SSM execution role to the SSM service. Finally it shows the FIS user needing to have a pass-role permission to grant the FIS execution role to the FIS service.</p>
 </div> 
</div> 
<p>This role needs to contain an assume role policy statement for FIS to allow assuming the role:</p> 
<pre><code class="lang-yaml">      AssumeRolePolicyDocument:
        Statement:
          - Action:
              - 'sts:AssumeRole'
            Effect: Allow
            Principal:
              Service:
                - "fis.amazonaws.com"</code></pre> 
<p>The role also needs permissions to list and execute SSM documents:</p> 
<pre><code class="lang-yaml">            - Sid: RequiredReadActionsforAWSFIS
              Effect: Allow
              Action:
                - 'cloudwatch:DescribeAlarms'
                - 'ssm:GetAutomationExecution'
                - 'ssm:ListCommands'
                - 'iam:ListRoles'
              Resource: '*'
            - Sid: RequiredSSMStopActionforAWSFIS
              Effect: Allow
              Action:
                - 'ssm:CancelCommand'
              Resource: '*'
            - Sid: RequiredSSMWriteActionsforAWSFIS
              Effect: Allow
              Action:
                - 'ssm:StartAutomationExecution'
                - 'ssm:StopAutomationExecution'
              Resource: 
                - !Sub 'arn:aws:ssm:${AWS::Region}:${AWS::AccountId}:automation-definition/${SsmAutomationIamAttachDetachDocument}:$DEFAULT'</code></pre> 
<p>Finally, remember that the SSM document needs to use a Role of its own to execute the fault injection actions. Because that Role is different from the Role under which we started the FIS experiment, we need to explicitly allow SSM to assume that role with a PassRole statement which will expand to&nbsp;<code>FISAPI-SSM-Automation-Role</code>:</p> 
<pre><code class="lang-yaml">            - Sid: RequiredIAMPassRoleforSSMADocuments
              Effect: Allow
              Action: 'iam:PassRole'
              Resource: !Sub 'arn:aws:iam::${AWS::AccountId}:role/${SsmAutomationRole}'</code></pre> 
<h2 id="temp:C:bBce9bb8c91650f41c7962af2e3e"><b>Secure and flexible permissions</b></h2> 
<p>So far, we have used explicit ARNs for our guardrails. To expand flexibility, we can use wildcards in our resource matching. For example, we might change the Policy matching from:</p> 
<pre><code class="lang-yaml">            Condition:
              ArnEquals:
                'iam:PolicyARN':
                  # Explicitly listed policies - secure but inflexible
                  - !Ref AwsFisApiPolicyDenyS3DeleteObject</code></pre> 
<p>or the equivalent:</p> 
<pre><code class="lang-yaml">            Condition:
              ArnEquals:
                'iam:PolicyARN':
                  # Explicitly listed policies - secure but inflexible
                  - !Sub 'arn:${AWS::Partition}:iam::${AWS::AccountId}:policy/${FullPolicyName}</code></pre> 
<p>to a wildcard notation like this:</p> 
<pre><code class="lang-yaml">            Condition:
              ArnEquals:
                'iam:PolicyARN':
                  # Wildcard policies - secure and flexible
                  - !Sub 'arn:${AWS::Partition}:iam::${AWS::AccountId}:policy/${PolicyNamePrefix}*'</code></pre> 
<p>If we set <code>PolicyNamePrefix</code>&nbsp;to <code>FISAPI-DenyS3</code>&nbsp;this would now allow invoking <code>FISAPI-DenyS3PutObject</code>&nbsp;and <code>FISAPI-DenyS3DeleteObject</code>&nbsp;but would not allow using a policy named <code>FISAPI-DenyEc2DescribeInstances</code>.</p> 
<p>Similarly, we could change the Resource matching from:</p> 
<pre><code class="lang-yaml">            NotResource: 
              # Explicitly listed roles - secure but inflexible
              - !GetAtt EventBridgeTimerHandlerRole.Arn
              - !GetAtt S3TriggerHandlerRole.Arn</code></pre> 
<p>to a wildcard equivalent like this:</p> 
<pre><code class="lang-yaml">            NotResource: 
              # Wildcard policies - secure and flexible
              - !Sub 'arn:${AWS::Partition}:iam::${AWS::AccountId}:role/${RoleNamePrefixEventBridge}*'
              - !Sub 'arn:${AWS::Partition}:iam::${AWS::AccountId}:role/${RoleNamePrefixS3}*'</code></pre> 
<pre id="temp:C:bBcd609ee79be8f49279a9ec92c9">and setting <code>RoleNamePrefixEventBridge</code>&nbsp;to <code>FISAPI-TARGET-EventBridge</code>&nbsp;and <code>RoleNamePrefixS3</code>&nbsp;to <code>FISAPI-TARGET-S3</code>.</pre> 
<p>Finally, we would also change the FIS experiment role to allow SSM documents based on a name prefix by changing the constraint on automation execution from:</p> 
<pre><code class="lang-yaml">            - Sid: RequiredSSMWriteActionsforAWSFIS
              Effect: Allow
              Action:
                - 'ssm:StartAutomationExecution'
                - 'ssm:StopAutomationExecution'
              Resource: 
                # Explicitly listed resource - secure but inflexible
                # Note: the $DEFAULT at the end could also be an explicit version number
                # Note: the 'automation-definition' is automatically created from 'document' on invocation
                - !Sub 'arn:aws:ssm:${AWS::Region}:${AWS::AccountId}:automation-definition/${SsmAutomationIamAttachDetachDocument}:$DEFAULT'</code></pre> 
<p id="temp:C:bBcc627d9cc1e594ff6a5ee9210c">to</p> 
<pre><code class="lang-yaml">            - Sid: RequiredSSMWriteActionsforAWSFIS
              Effect: Allow
              Action:
                - 'ssm:StartAutomationExecution'
                - 'ssm:StopAutomationExecution'
              Resource: 
                # Wildcard resources - secure and flexible
                # 
                # Note: the 'automation-definition' is automatically created from 'document' on invocation
                - !Sub 'arn:aws:ssm:${AWS::Region}:${AWS::AccountId}:automation-definition/${SsmAutomationDocumentPrefix}*'</code></pre> 
<p>and setting&nbsp;<code>SsmAutomationDocumentPrefix</code>&nbsp;to <code>FISAPI-</code>. Test this by updating the CloudFormation stack with a modified <a href="https://github.com/aws-samples/fis-api-failure-injection-using-iam/blob/main/template2.yaml">template</a>:</p> 
<pre><code class="lang-bash">aws cloudformation deploy --stack-name test-fis-api-faults --template-file template2.yaml --capabilities CAPABILITY_NAMED_IAM</code></pre> 
<h2 id="temp:C:bBc62a44308979349efb6cb6a8a2">Permissions governing users</h2> 
<p>In production you should not be using administrator access to use FIS. Instead we create two roles <code>FISAPI-AssumableRoleWithCreation</code>&nbsp;and <code>FISAPI-AssumableRoleWithoutCreation</code>&nbsp;for you (see this <a href="https://github.com/aws-samples/fis-api-failure-injection-using-iam/blob/main/template2.yaml">template</a>). These roles require all FIS and SSM resources to have a <code>Name</code>&nbsp;tag that starts with <code>FISAPI-</code>. Try <a href="https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_use_switch-role-console.html">assuming</a>&nbsp;the role without creation privileges and running an experiment. You will notice that you can only start an experiment if you add a <code>Name</code>&nbsp;tag, e.g. <code>FISAPI-secure-1</code>, and you will only be able to get details of experiments and templates that have proper <code>Name</code>&nbsp;tags.</p> 
<p>If you are working with <a href="https://docs.aws.amazon.com/organizations/latest/userguide/orgs_introduction.html">AWS Organizations</a>, you can add further guard rails by defining SCPs that control the use of the <code>FISAPI-*</code>&nbsp;tags similar to this <a href="https://aws.amazon.com/blogs/security/securing-resource-tags-used-for-authorization-using-service-control-policy-in-aws-organizations/">blog post</a>.</p> 
<h2 id="temp:C:bBcddb6f44a6d3c4703aeca4628e"><b>Caveats</b></h2> 
<p>For this solution we are choosing to attach policies instead of permission boundaries. The benefit of this is that you can attach multiple independent policies and thus simulate multi-step service degradation. However, this means that it is possible to <i>increase</i>&nbsp;the permission level of a role. While there are situations where this might be of interest, e.g. to simulate security breaches, please implement a thorough security review of any fault injection IAM policies you create. Note that modifying IAM Roles may trigger events in your security monitoring tools.</p> 
<p>The <code>AttachRolePolicy</code>&nbsp;and <code>DetachRolePolicy</code> calls from AWS IAM are eventually consistent, meaning that in some cases permission propagation when starting and stopping fault injection may take up to 5 minutes each.</p> 
<h2 id="temp:C:bBcab9bbc3b910743438ea68ed42"><b>Cleanup</b></h2> 
<p>To avoid additional cost, delete the content of the S3 bucket and delete the CloudFormation stack:</p> 
<pre><code class="lang-bash"># Clean up policy attachments just in case
CLEANUP_ROLES=$(aws iam list-roles --query "Roles[?starts_with(RoleName,'FISAPI-')].RoleName" --output text)
for role in $CLEANUP_ROLES; do
  CLEANUP_POLICIES=$(aws iam list-attached-role-policies --role-name $role --query "AttachedPolicies[?starts_with(PolicyName,'FISAPI-')].PolicyName" --output text)
  for policy in $CLEANUP_POLICIES; do
    echo Detaching policy $policy from role $role
    aws iam detach-role-policy --role-name $role --policy-arn $policy
  done
done
# Delete S3 bucket content
ACCOUNT_ID=$( aws sts get-caller-identity --query Account --output text )
S3_BUCKET_NAME=fis-api-failure-${ACCOUNT_ID}
aws s3 rm --recursive s3://${S3_BUCKET_NAME}
aws s3 rb s3://${S3_BUCKET_NAME}
# Delete cloudformation stack
aws cloudformation delete-stack --stack-name test-fis-api-faults
aws cloudformation wait stack-delete-complete --stack-name test-fis-api-faults</code></pre> 
<h2 id="temp:C:bBc163e2e0551944362b98203085"><b>Conclusion&nbsp;</b></h2> 
<p>AWS Fault Injection Simulator provides the ability to simulate various external impacts to your application to validate and improve resilience. We’ve shown how combining FIS with IAM to selectively deny access to AWS APIs provides&nbsp;a generic path to explore fault boundaries across all AWS services. We’ve shown how this can be used to identify and improve a resilience problem in a common S3 upload workflow. To learn about more ways to use FIS, see this&nbsp;<a href="https://chaos-engineering.workshop.aws/">workshop</a>.</p> 
<p><strong>About the authors:</strong></p> 
<footer> 
 <div class="blog-author-box"> 
  <div class="blog-author-image">
   <img alt="" class="aligncenter size-full wp-image-11636" height="160" src="https://d2908q01vomqb2.cloudfront.net/7719a1c782a1ba91c031a682a0a2f8658209adbf/2023/02/23/fis-iam-api-rudolf-potucek.png" width="120" />
  </div> 
  <h3 class="lb-h4">Dr. Rudolf Potucek</h3> 
  <p>Dr. Rudolf Potucek is Startup Solutions Architect at Amazon Web Services. Over the past 30 years he gained a PhD and worked in different roles including leading teams in academia and industry, as well as consulting. He brings experience from working with academia, startups, and large enterprises to his current role of guiding startup customers to succeed in the cloud.</p> 
 </div> 
</footer> 
<footer> 
 <div class="blog-author-box"> 
  <div class="blog-author-image">
   <img alt="" class="aligncenter wp-image-11636" height="160" src="https://d2908q01vomqb2.cloudfront.net/7719a1c782a1ba91c031a682a0a2f8658209adbf/2023/02/23/fis-iam-api-rudolph-wagner.png" width="120" />
  </div> 
  <h3 class="lb-h4">Rudolph Wagner</h3> 
  <p>Rudolph Wagner is a Premium Support Engineer at Amazon Web Services who holds the CISSP and OSCP security certifications, in addition to being a certified AWS Solutions Architect Professional. He assists internal and external Customers with multiple AWS services by using his diverse background in SAP, IT, and construction.</p> 
 </div> 
</footer>
