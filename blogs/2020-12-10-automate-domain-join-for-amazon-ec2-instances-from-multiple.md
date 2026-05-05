---
title: "Automate domain join for Amazon EC2 instances from multiple AWS accounts and Regions"
url: "https://aws.amazon.com/blogs/security/automate-domain-join-for-amazon-ec2-instances-multiple-aws-accounts-regions/"
date: "Thu, 10 Dec 2020 19:22:35 +0000"
author: "Sanjay Patel"
feed_url: "https://aws.amazon.com/blogs/security/tag/aws-directory-service/feed/"
---
<p>As organizations scale up their <a href="http://aws.amazon.com/" rel="noopener noreferrer" target="_blank">Amazon Web Services (AWS)</a> presence, they are faced with the challenge of administering user identities and controlling access across multiple accounts and Regions. As this presence grows, managing user access to cloud resources such as <a href="https://aws.amazon.com/ec2/" rel="noopener noreferrer" target="_blank">Amazon Elastic Compute Cloud (Amazon EC2)</a> becomes increasingly complex. <a href="https://aws.amazon.com/directoryservice/active-directory/" rel="noopener noreferrer" target="_blank">AWS Directory Service for Microsoft Active Directory</a> (also known as an AWS Managed Microsoft AD) makes it easier and more cost-effective for you to manage this complexity. AWS Managed Microsoft AD is built on highly available, AWS managed infrastructure. Each directory is deployed across multiple Availability Zones, and monitoring automatically detects and replaces domain controllers that fail. In addition, data replication and automated daily snapshots are configured for you. You don’t have to install software, and AWS handles all patching and software updates. AWS Managed Microsoft AD enables you to leverage your existing on-premises user credentials to access cloud resources such as the AWS Management Console and <a href="https://aws.amazon.com/ec2/" rel="noopener noreferrer" target="_blank">EC2</a> instances.</p> 
<p>This blog post describes how EC2 resources launched across multiple AWS accounts and Regions can automatically domain-join a centralized AWS Managed Microsoft AD. The solution we describe in this post is implemented for both Windows and Linux instances. Removal of Computer objects from Active Directory upon instance termination is also implemented. The solution uses <a href="https://aws.amazon.com/dynamodb/" rel="noopener noreferrer" target="_blank">Amazon DynamoDB</a> to centrally store account and directory information in a central security account. We also provide <a href="https://aws.amazon.com/cloudformation/" rel="noopener noreferrer" target="_blank">AWS CloudFormation</a> templates and platform-specific domain join scripts for you to use with <a href="https://aws.amazon.com/lambda/" rel="noopener noreferrer" target="_blank">AWS Lambda</a> as a quick start solution.</p> 
<h2>Architecture</h2> 
<p>The following diagram shows the domain-join process for EC2 instances across multiple accounts and Regions using <a href="https://aws.amazon.com/directoryservice/" rel="noopener noreferrer" target="_blank">AWS Managed Microsoft AD</a>.</p> 
<div class="wp-caption aligncenter" id="attachment_17537" style="width: 1800px;">
 <img alt="Figure 1: EC2 domain join architecture" class="size-full wp-image-17537" height="1154" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2020/11/27/Automate-domain-join-EC2-Figure-1.png" width="1790" />
 <p class="wp-caption-text" id="caption-attachment-17537">Figure 1: EC2 domain join architecture</p>
</div> 
<p>The event flow works as follows:</p> 
<ol> 
 <li>An EC2 instance is launched in a <a href="https://docs.aws.amazon.com/vpc/latest/peering/what-is-vpc-peering.html" rel="noopener noreferrer" target="_blank">peered virtual private cloud (VPC)</a> of a workload or security account. VPCs that are hosting EC2 instances need to be peered with the VPC that contains AWS Managed Microsoft AD to enable network connectivity with Active Directory.</li> 
 <li>An <a href="https://docs.aws.amazon.com/AmazonCloudWatch/latest/events/WhatIsCloudWatchEvents.html" rel="noopener noreferrer" target="_blank">Amazon CloudWatch Events</a> rule detects an EC2 instance in the “running” state.</li> 
 <li>The CloudWatch event is forwarded to a regional CloudWatch event bus in the security account.</li> 
 <li>If the CloudWatch event bus is in the same Region as AWS Managed Microsoft AD, it delivers the event to an <a href="https://aws.amazon.com/sqs/" rel="noopener noreferrer" target="_blank">Amazon Simple Queue Service (Amazon SQS)</a> queue, referred to as the <em>domain-join queue</em> in this post.</li> 
 <li>If the CloudWatch event bus is in a different Region from AWS Managed Microsoft AD, it delivers the event to an <a href="https://aws.amazon.com/sns/" rel="noopener noreferrer" target="_blank">Amazon Simple Notification Service (Amazon SNS)</a> topic. The event is then delivered to the domain-join queue described in step 4, through the Amazon SNS topic subscription.</li> 
 <li>Messages in the domain-join queue are held for five minutes to allow for EC2 instances to stabilize after they reach the “running” state. This delay allows time for installation of additional software components and agents through the use of EC2 <a href="https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/user-data.html" rel="noopener noreferrer" target="_blank">user data</a> and <a href="https://docs.aws.amazon.com/systems-manager/latest/userguide/distributor.html" rel="noopener noreferrer" target="_blank">AWS Systems Manager Distributor</a>.</li> 
 <li>After the holding period is over, messages in the domain-join queue invoke the AWS AD Join/Leave <a href="https://aws.amazon.com/lambda/" rel="noopener noreferrer" target="_blank">Lambda</a> function. The Lambda function does the following: 
  <ol type="a"> 
   <li>Retrieves the AWS account ID that originated the event from the message and retrieves account-specific configurations from a DynamoDB table. This configuration identifies AWS Managed Microsoft AD domain controller IPs, credentials required to perform EC2 domain join, and an <a href="http://aws.amazon.com/iam" rel="noopener noreferrer" target="_blank">AWS Identity and Access Management (IAM)</a> role that can be assumed by the Lambda function to invoke <a href="https://docs.aws.amazon.com/systems-manager/latest/userguide/execute-remote-commands.html" rel="noopener noreferrer" target="_blank">AWS Systems Manager Run Command</a>.</li> 
   <li>If needed, uses <a href="https://aws.amazon.com/iam/" rel="noopener noreferrer" target="_blank">AWS Security Token Service (AWS STS)</a> and prepares a cross-account access session.</li> 
   <li>Retrieves EC2 instance information, such as the instance state, platform, and tags, and validates the instance state.</li> 
   <li>Retrieves platform-specific domain-join scripts that are deployed with the Lambda function’s code bundle, and configures invocation of those scripts by using data read from the DynamoDB table (bash script for Linux instances and PowerShell script for Windows instances).</li> 
   <li>Uses AWS Systems Manager Run Command to invoke the domain-join script on the instance. Run Command enables you to remotely and securely manage the configuration of your managed instances.</li> 
   <li>The domain-join script runs on the instance. It uses script parameters and instance attributes to configure the instance and perform the domain join. The adGroupName tag value is used to configure the Active Directory user group that will have permissions to log in to the instance. The instance is rebooted to complete the domain join process. Various software components are installed on the instance when the script runs. For the Linux instance, sssd, realmd, krb5, samba-common, adcli, unzip, and packageit are installed. For the Windows instance, the RDS-RD-Server feature is installed.</li> 
  </ol> </li> 
</ol> 
<p>Removal of EC2 instances from AWS Managed Microsoft AD upon instance termination follows a similar sequence of steps. Each instance that is domain joined creates an Active Directory domain object under the “Computer” hierarchy. This domain object needs to be removed upon instance termination so that a new instance that uses the same private IP address in the subnet (at a future time) can successfully domain join and enable instance access with Active Directory credentials. Removal of the Active Directory Computer object is done by running the leaveDomaini.ps1 script (included with this blog) through Run Command on the Active Directory Tools instance identified in Figure 1.</p> 
<h2>Prerequisites and setup</h2> 
<p>To build the solution outlined in this post, you need:</p> 
<ul> 
 <li><a href="https://aws.amazon.com/directoryservice/active-directory/" rel="noopener noreferrer" target="_blank">AWS Managed Microsoft AD</a> with an appropriate DNS name (for example, example.com). For more information about getting started with AWS Managed Microsoft AD, see <a href="https://docs.aws.amazon.com/directoryservice/latest/admin-guide/ms_ad_getting_started_create_directory.html" rel="noopener noreferrer" target="_blank">Create Your AWS Managed Microsoft AD directory</a>.</li> 
 <li>AD Tools. To install AD Tools and use it to create the required users: 
  <ol> 
   <li>Launch a Windows EC2 instance in the same account and Region, and domain-join it with the directory you created in the previous step. Log in to the instance through Remote Desktop Protocol (RDP) and install AD Tools as described in <a href="https://docs.aws.amazon.com/directoryservice/latest/admin-guide/ms_ad_install_ad_tools.html" rel="noopener noreferrer" target="_blank">Installing the Active Directory Administration Tools</a>.</li> 
   <li>After the AD Tools are installed, launch the AD Users &amp; Computers application to create domain users, and assign those users to an Active Directory security group (for example, my_UserGroup) that has permission to access domain-joined instances.</li> 
   <li>Create a least-privileged user for performing domain joins as described in <a href="https://docs.aws.amazon.com/directoryservice/latest/admin-guide/directory_join_privileges.html" rel="noopener noreferrer" target="_blank">Delegate Directory Join Privileges for AWS Managed Microsoft AD</a>. The identity of this user is stored in the DynamoDB table and read by the AD Join Lambda function to invoke Active Directory join scripts.</li> 
   <li>Store the password for the least-privileged user in an encrypted <a href="https://docs.aws.amazon.com/systems-manager/latest/userguide/systems-manager-parameter-store.html" rel="noopener noreferrer" target="_blank">Systems Manager parameter</a>. The password for this user is stored in the secure string System Manager parameter and read by the AD Join Lambda function at runtime while processing <a href="https://aws.amazon.com/sqs/" rel="noopener noreferrer" target="_blank">Amazon SQS</a> messages.</li> 
   <li>Assign a unique tag key and value to identify the AD Tools instance. This instance will be invoked by the Lambda function to delete Computer objects from Active Directory upon termination of domain-joined instances.</li> 
  </ol> </li> 
 <li>All VPCs that are hosting EC2 instances to be domain joined must be peered with the VPC that hosts the relevant AWS Managed Microsoft AD. Alternatively, <a href="https://aws.amazon.com/transit-gateway/" rel="noopener noreferrer" target="_blank">AWS Transit Gateway</a> could be used to establish this connectivity.</li> 
 <li>In addition to having network connectivity to the AWS Managed Microsoft AD domain controllers, domain join scripts that run on EC2 instances must be able to resolve relevant Active Directory resource records. In this solution, we leverage <a href="https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/resolver-getting-started.html" rel="noopener noreferrer" target="_blank">Amazon Route 53 Outbound Resolver</a> to forward DNS queries to the AWS Managed Microsoft AD DNS servers, while still preserving the default DNS capabilities that are available to the VPC. Learn more about <a href="https://aws.amazon.com/premiumsupport/knowledge-center/route53-resolve-with-outbound-endpoint/" rel="noopener noreferrer" target="_blank">deploying Route 53 Outbound Resolver and resolver rules to resolve your directory DNS name to DNS IPs</a>.</li> 
 <li>Each domain-join EC2 instance must have a Systems Manager Agent (SSM Agent) installed and an IAM role that provides equivalent permissions as provided by the AmazonEC2RoleforSSM built-in policy. The SSM Agent is used to allow domain-join scripts to run automatically. See <a href="https://docs.aws.amazon.com/systems-manager/latest/userguide/ssm-agent.html" rel="noopener noreferrer" target="_blank">Working with SSM Agent</a> for more information on installing and configuring SSM Agents on EC2 instances.</li> 
</ul> 
<h2>Solution deployment</h2> 
<p>The steps in this section deploy AD Join solution components by using the <a href="https://aws.amazon.com/cloudformation/" rel="noopener noreferrer" target="_blank">AWS CloudFormation</a> service.</p> 
<p>The CloudFormation template provided with this solution (<a href="https://aws-security-blog-content.s3.amazonaws.com/public/sample/520-Automate-domain-join-Amazon-EC2-instances+/mad_auto_join_leave.json" rel="noopener noreferrer" target="_blank">mad_auto_join_leave.json</a>) deploys resources that are identified in the security account’s AWS Region that hosts AWS Managed Microsoft AD (the top left quadrant highlighted in Figure 1). The template deploys a DynamoDB resource with 5 read and 5 write capacity units. This should be adjusted to match your usage. DynamoDB also provides the ability to auto-scale these capacities. You will need to create and deploy additional CloudFormation stacks for cross-account, cross-Region scenarios.</p> 
<h3>To deploy the solution</h3> 
<ol> 
 <li>Create a versioned <a href="https://aws.amazon.com/s3/" rel="noopener noreferrer" target="_blank">Amazon Simple Storage Service (Amazon S3)</a> bucket to store a zip file (for example, <a href="https://aws-security-blog-content.s3.amazonaws.com/public/sample/520-Automate-domain-join-Amazon-EC2-instances+/adJoinCode.zip" rel="noopener noreferrer" target="_blank">adJoinCode.zip</a>) that contains Python Lambda code and domain join/leave bash and PowerShell scripts. Upload the source code zip file to an S3 bucket and find the version associated with the object.</li> 
 <li>Navigate to the AWS CloudFormation console. Choose the appropriate AWS Region, and then choose <strong>Create Stack</strong>. Select <strong>With new resources</strong>.</li> 
 <li>Choose <strong>Upload a template file</strong> (for this solution, <a href="https://aws-security-blog-content.s3.amazonaws.com/public/sample/520-Automate-domain-join-Amazon-EC2-instances+/mad_auto_join_leave.json" rel="noopener noreferrer" target="_blank">mad_auto_join_leave.json</a>), select the CloudFormation stack file, and then choose <strong>Next</strong>.</li> 
 <li>Enter the stack name and values for the other parameters, and then choose <strong>Next</strong>. 
  <div class="wp-caption aligncenter" id="attachment_17540" style="width: 1056px;">
   <img alt="Figure 2: Defining the stack name and parameters" class="size-full wp-image-17540" height="1128" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2020/11/28/Automate-domain-join-EC2-Figure-2.png" width="1046" />
   <p class="wp-caption-text" id="caption-attachment-17540">Figure 2: Defining the stack name and parameters</p>
  </div> <p>The parameters are defined as follows:</p></li> 
</ol> 
<ul> 
 <li><strong>S3CodeBucket</strong>: The name of the S3 bucket that holds the Lambda code zip file object.</li> 
 <li><strong>adJoinLambdaCodeFileName</strong>: The name of the Lambda code zip file that includes Lambda Python code, bash, and Powershell scripts.</li> 
 <li><strong>adJoinLambdaCodeVersion</strong>: The S3 Version ID of the uploaded Lambda code zip file.</li> 
 <li><strong>DynamoDBTableName</strong>: The name of the DynamoDB table that will hold account configuration information.</li> 
 <li><strong>CreateDynamoDBTable</strong>: The flag that indicates whether to create a new DynamoDB table or use an existing table.</li> 
 <li><strong>ADToolsHostTagKey</strong>: The tag key of the Windows EC2 instance that has AD Tools installed and that will be used for removal of Active Directory Computer objects upon instance termination.</li> 
 <li><strong>ADToolsHostTagValue</strong>: The tag value for the key identified by the ADToolsHostTagKey parameter.</li> 
 <li>Acknowledge creation of AWS resources and choose to continue to deploy AWS resources through AWS CloudFormation.The CloudFormation stack creation process is initiated, and after a few minutes, upon completion, the stack status is marked as CREATE_COMPLETE. The following resources are created when the CloudFormation stack deploys successfully: 
  <ul> 
   <li>An AD Join Lambda function with associated scripts and IAM role.</li> 
   <li>A <a href="https://docs.aws.amazon.com/AmazonCloudWatch/latest/events/WhatIsCloudWatchEvents.html" rel="noopener noreferrer" target="_blank">CloudWatch Events</a> rule to detect the “running” and “terminated” states for EC2 instances.</li> 
   <li>An <a href="https://aws.amazon.com/sqs/" rel="noopener noreferrer" target="_blank">SQS</a> event queue to hold the EC2 instance “running” and “terminated” events.</li> 
   <li>CloudWatch event mapping to the SQS event queue and further to the Lambda function.</li> 
   <li>A <a href="https://aws.amazon.com/dynamodb/" rel="noopener noreferrer" target="_blank">DynamoDB</a> table to hold the account configuration (if you chose this option).</li> 
  </ul> </li> 
</ul> 
<p>The DynamoDB table hosts account-level configurations. Account-specific configuration is required for an instance from a given account to join the Active Directory domain. Each DynamoDB item contains the account-specific configuration shown in the following table. Storing account-level information in the DynamoDB table provides the ability to use multiple AWS Managed Microsoft AD directories and group various accounts accordingly. Additional account configurations can also be stored in this table for implementation of various centralized security services (instance inspection, patch management, and so on).</p> 
<table border="1" style="margin-left: 45px;" width="0"> 
 <tbody> 
  <tr> 
   <td style="padding-left: 10px;" width="233"><strong>Attribute</strong></td> 
   <td style="padding-left: 10px;" width="301"><strong>Description</strong></td> 
  </tr> 
  <tr style="padding-left: 10px;"> 
   <td style="padding-left: 10px;" width="233">accountId</td> 
   <td style="padding-left: 10px;" width="301">AWS account number</td> 
  </tr> 
  <tr style="padding-left: 10px;"> 
   <td style="padding-left: 10px;" width="233">adJoinUserName</td> 
   <td style="padding-left: 10px;" width="301">User ID with AD Join permissions</td> 
  </tr> 
  <tr style="padding-left: 10px;"> 
   <td style="padding-left: 10px;" width="233">adJoinUserPwParam</td> 
   <td style="padding-left: 10px;" width="301">Encrypted Systems Manager parameter containing the AD Join user’s password</td> 
  </tr> 
  <tr style="padding-left: 10px;"> 
   <td style="padding-left: 10px;" width="233">dnsIP1</td> 
   <td style="padding-left: 10px;" width="301">Domain controller 1 IP address2</td> 
  </tr> 
  <tr style="padding-left: 10px;"> 
   <td style="padding-left: 10px;" width="233">dnsIP2</td> 
   <td style="padding-left: 10px;" width="301">Domain controller 2 IP address</td> 
  </tr> 
  <tr style="padding-left: 10px;"> 
   <td style="padding-left: 10px;" width="233">assumeRoleARN</td> 
   <td style="padding-left: 10px;" width="301">Amazon Resource Name (ARN) of the role assumed by the AD Join Lambda function</td> 
  </tr> 
 </tbody> 
</table> 
<p>Following is an example of how you could insert an item (row) in a DynamoDB table for an account.</p> 
<div class="hide-language"> 
 <pre><code class="lang-text">aws dynamodb put-item --table-name <em><span style="font-family: courier;"><span style="color: #ff0000;">&lt;DynamoDB-Table-Name&gt;</span></span></em> --item file://itemData.json
</code></pre> 
</div> 
<p>where itemData.json is as follows.</p> 
<div class="hide-language"> 
 <pre><code class="lang-text">{
    "accountId": { "S": "123412341234" },
    " adJoinUserName": { "S": "ADJoinUser" },
    " adJoinUserPwParam": { "S": "ADJoinUser-PwParam" },
    "dnsName": { "S": "example.com" },
    "dnsIP1": { "S": "192.0.2.1" },
    "dnsIP2": { "S": "192.0.2.2" },
    "assumeRoleARN": { "S": "arn:aws:iam::111122223333:role/adJoinLambdaRole" }
}
</code></pre> 
</div> 
<p>(Update with your own values as appropriate for your environment.)</p> 
<p>In the preceding example, adJoinLambdaRole is assumed by the AD Join Lambda function (if needed) to establish cross-account access using <a href="https://aws.amazon.com/iam/" rel="noopener noreferrer" target="_blank">AWS Security Token Service (AWS STS)</a>. The role needs to provide sufficient privileges for the AD Join Lambda function to retrieve instance information and run cross-account Systems Manager commands.</p> 
<p>adJoinUserName identifies a user with the minimum privileges to do the domain join; you created this user in the prerequisite steps.</p> 
<p>adJoinUserPwParam identifies the name of the encrypted Systems Manager parameter that stores the password for the AD Join user. You created this parameter in the prerequisite steps.</p> 
<h2>Solution test</h2> 
<p>After you successfully deploy the solution using the steps in the previous section, the next step is to test the deployed solution.</p> 
<h3>To test the solution</h3> 
<ol> 
 <li>Navigate to the AWS EC2 console and launch a Linux instance. Launch the instance in a public subnet of the available VPC.</li> 
 <li>Choose an IAM role that gives at least AmazonEC2RoleforSSM permissions to the instance.</li> 
 <li>Add an adGroupName tag with the value that identifies the name of the Active Directory security group whose members should have access to the instance.</li> 
 <li>Make sure that the security group associated with your instance has permissions for your IP address to log in to the instance by using the Secure Shell (SSH) protocol.</li> 
 <li>Wait for the instance to launch and perform the Active Directory domain join. You can navigate to the AWS SQS console and observe a delayed message that represents the CloudWatch instance “running” event. This message is processed after five minutes; after that you can observe the Lambda function’s message processing log in CloudWatch logs.</li> 
 <li>Log in to the instance with Active Directory user credentials. This user must be the member of the Active Directory security group identified by the adGroupName tag value. Following is an example login command. 
  <div class="hide-language"> 
   <pre><code class="lang-text">ssh ‘user1@example.com’@<em><span style="font-family: courier;"><span style="color: #ff0000;">&lt;public-dns-name|public-ip-address&gt;</span></span></em>
</code></pre> 
  </div> </li> 
 <li>Similarly, launch a Windows EC2 instance to validate the Active Directory domain join by using Remote Desktop Protocol (RDP).</li> 
 <li>Terminate domain-joined instances. Log in to the AD Tools instance to validate that the Active Directory Computer object that represents the instance is deleted.</li> 
</ol> 
<p>The AD Join Lambda function invokes <a href="https://docs.aws.amazon.com/systems-manager/latest/userguide/execute-remote-commands.html" rel="noopener noreferrer" target="_blank">Systems Manager commands</a> to deliver and run domain join scripts on the EC2 instances. The <span style="font-family: courier;">AWS-RunPowerShellScript</span> command is used for Microsoft Windows instances, and the <span style="font-family: courier;">AWS-RunShellScript</span> command is used for Linux instances. Systems Manager command parameters and execution status can be observed in the Systems Manager Run Command console.</p> 
<p>The AD user used to perform the domain join is a least-privileged user, as described in <a href="https://docs.aws.amazon.com/directoryservice/latest/admin-guide/directory_join_privileges.html" rel="noopener noreferrer" target="_blank">Delegate Directory Join Privileges for AWS Managed Microsoft AD.</a> The password for this user is passed to instances by way of SSM Run Commands, as described above. The password is visible in the SSM Command history log and in the domain join scripts run on the instance. Alternatively, all script parameters can be read locally on the instance through the “adjoin” encrypted SSM parameter. Refer to the domain join scripts for details of the “adjoin” SSM parameter.</p> 
<h2>Additional information</h2> 
<h3>Directory sharing</h3> 
<p>AWS Managed Microsoft AD can be shared with other AWS accounts in the same Region. Learn how to use this feature and seamlessly domain join <a href="https://aws.amazon.com/blogs/security/how-to-domain-join-amazon-ec2-instances-aws-managed-microsoft-ad-directory-multiple-accounts-vpcs/" rel="noopener noreferrer" target="_blank">Microsoft Windows EC2 instances</a> and <a href="https://aws.amazon.com/blogs/aws/seamlessly-join-a-linux-instance-to-aws-directory-service-for-microsoft-active-directory/" rel="noopener noreferrer" target="_blank">Linux instances</a>.</p> 
<h3>autoadjoin tag</h3> 
<p>Launching EC2 instances with an autoadjoin tag key with a “false” value excludes the instance from the automated Active Directory join process. You might want to do this in scenarios where you want to install additional agent software before or after the Active Directory join process. You can invoke domain join scripts (bash or PowerShell) by using user data or other means. However, you’ll need to reboot the instance and re-run scripts to complete the domain join process.</p> 
<h2>Summary</h2> 
<p>In this blog post, we demonstrated how you could automate the Active Directory domain join process for EC2 instances to AWS Managed Microsoft AD across multiple accounts and Regions, and also centrally manage this configuration by using <a href="https://aws.amazon.com/dynamodb/" rel="noopener noreferrer" target="_blank">AWS DynamoDB</a>. By adopting this model, administrators can centrally manage Active Directory–aware applications and resources across their accounts.</p> 
<p>If you have feedback about this post, submit comments in the <strong>Comments</strong> section below. If you have questions about this post, <a href="https://console.aws.amazon.com/support/home" rel="noopener noreferrer" target="_blank" title="contact AWS Support">contact AWS Support</a>.</p> 
<p><strong>Want more AWS Security how-to content, news, and feature announcements? Follow us on <a href="https://twitter.com/AWSsecurityinfo" rel="noopener noreferrer" target="_blank" title="Twitter">Twitter</a>.</strong></p> 
<footer> 
 <div class="blog-author-box"> 
  <div class="blog-author-image">
   <img alt="Author" class="aligncenter size-full wp-image-16830" height="160" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2020/10/30/Sanjay-Patel-Author.jpg" width="120" />
  </div> 
  <h3 class="lb-h4">Sanjay Patel</h3> 
  <p>Sanjay is a Senior Cloud Application Architect with AWS Professional Services. He has a diverse background in software design, enterprise architecture, and API integrations. He has helped AWS customers automate infrastructure security. He enjoys working with AWS customers to identify and implement the best fit solution.</p> 
 </div> 
</footer> 
<footer> 
 <div class="blog-author-box"> 
  <div class="blog-author-image">
   <img alt="Author" class="aligncenter size-full wp-image-16829" height="160" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2020/10/30/Vaibhawa-Kuman-Author.jpg" width="120" />
  </div> 
  <h3 class="lb-h4">Vaibhawa Kumar</h3> 
  <p>Vaibhawa is a Senior Cloud Infrastructure Architect with AWS Professional Services. He helps customers with the architecture, design, and automation to build innovative, secured, and highly available solutions using various AWS services. In his free time, you can find him spending time with family, sports, and cooking.</p> 
 </div> 
</footer> 
<footer> 
 <div class="blog-author-box"> 
  <div class="blog-author-image">
   <img alt="Author" class="aligncenter size-full wp-image-17598" height="160" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2020/12/01/Kevin-Higgins-Author.jpg" width="120" />
  </div> 
  <h3 class="lb-h4">Kevin Higgins</h3> 
  <p>Kevin is a Senior Cloud Infrastructure Architect with AWS Professional Services. He helps customers with the architecture, design, and development of cloud-optimized infrastructure solutions. As a member of the Microsoft Global Specialty Practice, he collaborates with AWS field sales, training, support, and consultants to help drive AWS product feature roadmap and go-to-market strategies.</p> 
 </div> 
</footer>
