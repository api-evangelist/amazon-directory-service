---
title: "Secure and automated domain membership management for EC2 instances with no internet access"
url: "https://aws.amazon.com/blogs/security/secure-and-automated-domain-membership-management-for-ec2-instances-with-no-internet-access/"
date: "Mon, 15 Feb 2021 21:41:23 +0000"
author: "Rakesh Singh"
feed_url: "https://aws.amazon.com/blogs/security/tag/aws-directory-service/feed/"
---
<p>In this blog post, I show you how to deploy an automated solution that helps you fully automate the Active Directory join and unjoin process for <a href="http://aws.amazon.com/ec2" rel="noopener noreferrer" target="_blank">Amazon Elastic Compute Cloud (Amazon EC2)</a> instances that don’t have internet access.</p> 
<p>Managing Active Directory domain membership for <a href="http://aws.amazon.com/ec2" rel="noopener noreferrer" target="_blank">EC2</a> instances in <a href="http://aws.amazon.com/" rel="noopener noreferrer" target="_blank">Amazon Web Services (AWS)</a> Cloud is a typical use case for many organizations. In a dynamic environment that can grow and shrink multiple times in a day, adding and removing computer objects from an Active Directory domain is a critical task and is difficult to manage without automation.</p> 
<p>AWS <a href="https://docs.aws.amazon.com/directoryservice/latest/admin-guide/ms_ad_join_instance.html" rel="noopener noreferrer" target="_blank">seamless domain join</a> provides a secure and reliable option to <a href="https://docs.aws.amazon.com/directoryservice/latest/admin-guide/ms_ad_join_instance.html" rel="noopener noreferrer" target="_blank">join an EC2 instance to your AWS Directory Service for Microsoft Active Directory</a>. It’s a recommended approach for automating joining a <a href="https://docs.aws.amazon.com/directoryservice/latest/admin-guide/launching_instance.html" rel="noopener noreferrer" target="_blank">Windows</a> or <a href="https://docs.aws.amazon.com/directoryservice/latest/admin-guide/seamlessly_join_linux_instance.html" rel="noopener noreferrer" target="_blank">Linux EC2</a> instance to the <a href="https://docs.aws.amazon.com/directoryservice/latest/admin-guide/directory_microsoft_ad.html" rel="noopener noreferrer" target="_blank">AWS Managed Microsoft AD</a> or to an existing on-premises Active Directory using AD Connector, or a standalone Simple AD directory running in the AWS Cloud. This method requires your EC2 instances to have connectivity to the public <a href="https://aws.amazon.com/directoryservice/" rel="noopener noreferrer" target="_blank">AWS Directory Service</a> endpoints. At the time of writing, Directory Service doesn’t have PrivateLink endpoint support. This means you must allow traffic from your instances to the public Directory Service endpoints via an internet gateway, network address translation (NAT) device, virtual private network (VPN) connection, or <a href="http://aws.amazon.com/directconnect" rel="noopener noreferrer" target="_blank">AWS Direct Connect</a> connection.</p> 
<p>At times, your organization might require that any traffic between your VPC and Directory Service—or any other AWS service—not leave the Amazon network. That means launching EC2 instances in an <a href="http://aws.amazon.com/vpc" rel="noopener noreferrer" target="_blank">Amazon Virtual Private Cloud (Amazon VPC)</a> with no internet access and still needing to join and unjoin the instances from the Active Directory domain. Provided your instances have network connectivity to the directory DNS addresses, the simplest solution in this scenario is to run the domain join commands manually on the EC2 instances and enter the domain credentials directly. Though this process can be secure—as you don’t need to store or hardcode the credentials—it’s time consuming and becomes difficult to manage in a dynamic environment where EC2 instances are launched and terminated frequently.</p> 
<p><a href="https://docs.aws.amazon.com/vpc/latest/userguide/vpc-endpoints.html" rel="noopener noreferrer" target="_blank">VPC endpoints</a> enable private connections between your VPC and supported AWS services. Private connections enable you to privately access services by using private IP addresses. Traffic between your VPC and other AWS services doesn’t leave the Amazon network. Instances in your VPC don’t need public IP addresses to communicate with resources in the service.</p> 
<p>The solution in this blog post uses <a href="https://docs.aws.amazon.com/secretsmanager/latest/userguide/intro.html" rel="noopener noreferrer" target="_blank">AWS Secrets Manager</a> to store the domain credentials and VPC endpoints to enable private connection between your VPC and other AWS services. The solution described here can be used in the following scenarios:</p> 
<ol> 
 <li>Manage domain join and unjoin for EC2 instances that don’t have internet access.</li> 
 <li>Manage only domain unjoin if you’re already using seamless domain join provided by AWS, or any other method for domain joining.</li> 
 <li>Manage only domain join for EC2 instances that don’t have internet access.</li> 
</ol> 
<p>This solution uses <a href="https://aws.amazon.com/cloudformation/" rel="noopener noreferrer" target="_blank">AWS CloudFormation</a> to deploy the required resources in your AWS account based on your choice from the preceding scenarios.</p> 
<blockquote>
 <p><strong>Note</strong>: If your EC2 instances can access the internet, then we recommend using the seamless domain join feature and using scenario 2 to remove computers from the Active Directory domain upon instance termination.</p>
</blockquote> 
<p>The solution described in this blog post is designed to provide a secure, automated method for joining and unjoining EC2 instances to an on-premises or AWS Managed Microsoft AD domain. The solution is best suited for use cases where the EC2 instances don’t have internet connectivity and the seamless domain join option cannot be used.</p> 
<h2>How this solution works</h2> 
<p>This blog post includes a <a href="https://aws.amazon.com/cloudformation/" rel="noopener noreferrer" target="_blank">CloudFormation</a> template that you can use to deploy this solution. The CloudFormation stack provisions an EC2 Windows instance running in an <a href="https://aws.amazon.com/ec2/autoscaling/" rel="noopener noreferrer" target="_blank">Amazon EC2 Auto Scaling group</a> that acts as a worker and is responsible for joining and unjoining other EC2 instances from the Active Directory domain. The worker instance communicates with other required AWS services such as <a href="http://aws.amazon.com/s3" rel="noopener noreferrer" target="_blank">Amazon Simple Storage Service (Amazon S3)</a>, Secrets Manager, and <a href="http://aws.amazon.com/sqs" rel="noopener noreferrer" target="_blank">Amazon Simple Queue Service (Amazon SQS)</a> using VPC endpoints. The stack also creates all of the other resources needed for this solution to work.</p> 
<p>Figure 1 shows the domain join and unjoin workflow for EC2 instances in an AWS account.</p> 
<div class="wp-caption aligncenter" id="attachment_18843" style="width: 810px;">
 <img alt="Figure 1: Workflow for joining and unjoining an EC2 instance from a domain with full protection of Active Directory credentials" class="size-full wp-image-18843" height="498" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2021/02/14/Secure-automated-domain-EC2-1.png" width="800" />
 <p class="wp-caption-text" id="caption-attachment-18843">Figure 1: Workflow for joining and unjoining an EC2 instance from a domain with full protection of Active Directory credentials</p>
</div> 
<p>The event flow in Figure 1 is as follows:</p> 
<ol> 
 <li>An EC2 instance is launched or terminated in an account.</li> 
 <li>An <a href="https://docs.aws.amazon.com/AmazonCloudWatch/latest/events/WhatIsCloudWatchEvents.html" rel="noopener noreferrer" target="_blank">Amazon CloudWatch Events</a> rule detects if the EC2 instance is in <em>running</em> or <em>terminated</em> state.</li> 
 <li>The CloudWatch event triggers an <a href="http://aws.amazon.com/lambda" rel="noopener noreferrer" target="_blank">AWS Lambda</a> function that looks for the tag <span style="font-family: courier;"><span style="font-weight: bold;">JoinAD: true</span></span> to check if the instance needs to join or unjoin the Active Directory domain.</li> 
 <li>If the tag value is <span style="font-family: courier;"><span style="font-weight: bold;">true</span></span>, the Lambda function writes the instance details to an <a href="https://aws.amazon.com/sqs/" rel="noopener noreferrer" target="_blank">Amazon Simple Queue Service (Amazon SQS)</a> queue.</li> 
 <li>A standalone, highly secured EC2 instance acts as a worker and polls the Amazon SQS queue for new messages.</li> 
 <li>Whenever there’s a new message in the queue, the worker EC2 instance invokes scripts on the remote EC2 instance to add or remove the instance from the domain based on the instance operating system and state.</li> 
</ol> 
<p>In this solution, the security of the Active Directory credentials is enhanced by storing them in Secrets Manager. To secure the stored credentials, the solution uses resource-based policies to restrict the access to only intended users and roles.</p> 
<p>The credentials can only be fetched dynamically from the EC2 instance that’s performing the domain join and unjoin operations. Any access to that instance is further restricted by a <a href="https://docs.aws.amazon.com/IAM/latest/UserGuide/access_controlling.html" rel="noopener noreferrer" target="_blank">custom AWS Identity and Access Management (IAM) policy</a> created by the CloudFormation stack. The following policies are created by the stack to enhance security of the solution components.</p> 
<ol> 
 <li><a href="https://docs.aws.amazon.com/secretsmanager/latest/userguide/auth-and-access_resource-based-policies.html" rel="noopener noreferrer" target="_blank">Resource-based policies for Secrets Manager</a> to restrict all access to the stored secret to only specific IAM entities (such as the <span style="font-family: courier;">EC2 IAM</span> role).</li> 
 <li><a href="https://docs.aws.amazon.com/AmazonS3/latest/dev/using-iam-policies.html" rel="noopener noreferrer" target="_blank">An S3 bucket policy</a> to prevent unauthorized access to the Active Directory join and remove scripts that are stored in the S3 bucket.</li> 
 <li>The IAM role that’s used to fetch the credentials from Secrets Manager is restricted by a <a href="https://docs.aws.amazon.com/IAM/latest/UserGuide/access_controlling.html" rel="noopener noreferrer" target="_blank">custom IAM policy</a> and can only be assumed by the worker EC2 instance. This prevents every entity other than the worker instance from using that IAM role.</li> 
 <li>All API and console access to the worker EC2 instance is restricted by a <a href="https://docs.aws.amazon.com/IAM/latest/UserGuide/access_controlling.html" rel="noopener noreferrer" target="_blank">custom IAM policy</a> with an explicit deny.</li> 
 <li>A policy to deny all but the worker EC2 instance access to the credentials in Secrets Manager. With the worker EC2 instance doing the work, the EC2 instances that need to join the domain don’t need access to the credentials in Secrets Manager or to scripts in the S3 bucket.</li> 
</ol> 
<h2>Prerequisites and setup</h2> 
<p>Before you deploy the solution, you must complete the following in the AWS account and Region where you want to deploy the CloudFormation stack.</p> 
<ol> 
 <li><a href="https://docs.aws.amazon.com/directoryservice/latest/admin-guide/directory_microsoft_ad.html" rel="noopener noreferrer" target="_blank">AWS Managed Microsoft AD</a> with an appropriate DNS name (for example, <span style="font-family: courier;">test.com</span>). You can also use your on premises Active Directory, provided it’s reachable from the Amazon VPC over <a href="http://aws.amazon.com/directconnect" rel="noopener noreferrer" target="_blank">Direct Connect</a> or <a href="https://aws.amazon.com/vpn/" rel="noopener noreferrer" target="_blank">AWS VPN</a>.</li> 
 <li><a href="https://docs.aws.amazon.com/directoryservice/latest/admin-guide/simple_ad_dhcp_options_set.html" rel="noopener noreferrer" target="_blank">Create a DHCP option set</a> with on-premises DNS servers or with the DNS servers pointing to the IP addresses of directories provided by AWS.</li> 
 <li>Associate the DHCP option set with the Amazon VPC that you’re going to use with this solution.</li> 
 <li>Any other Amazon VPCs that are hosting EC2 instances to be domain joined must be peered with the VPC that hosts the relevant AWS Managed Microsoft AD. Alternatively, <a href="https://aws.amazon.com/transit-gateway/" rel="noopener noreferrer" target="_blank">AWS Transit Gateway</a> can be used to establish this connectivity.</li> 
 <li>Make sure to have the latest <a href="https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-install.html" rel="noopener noreferrer" target="_blank">AWS Command Line Interface (AWS CLI)</a> installed and configured on your local machine.</li> 
 <li>Create a new SSH key pair and store it in Secrets Manager using the following commands. Replace <em><span style="font-family: courier;"><span style="color: #ff0000;">&lt;Region&gt;</span></span></em> with the Region of your deployment. Replace <em><span style="font-family: courier;"><span style="color: #ff0000;">&lt;MyKeyPair&gt;</span></span></em> with any custom name or leave it default.</li> 
</ol> 
<p>Bash:</p> 
<div class="hide-language"> 
 <pre><code class="lang-text">aws ec2 create-key-pair --region <em><span style="font-family: courier;"><span style="color: #ff0000;">&lt;Region&gt;</span></span></em> --key-name <em><span style="font-family: courier;"><span style="color: #ff0000;">&lt;MyKeyPair&gt;</span></span></em> --query 'KeyMaterial' --output text &gt; adsshkey
aws secretsmanager create-secret --region <em><span style="font-family: courier;"><span style="color: #ff0000;">&lt;Region&gt;</span></span></em> --name "adsshkey" --description "my ssh key pair" --secret-string file://adsshkey
</code></pre> 
</div> 
<p>PowerShell:</p> 
<div class="hide-language"> 
 <pre><code class="lang-text">aws ec2 create-key-pair --region <em><span style="font-family: courier;"><span style="color: #ff0000;">&lt;Region&gt;</span></span></em> --key-name <em><span style="font-family: courier;"><span style="color: #ff0000;">&lt;MyKeyPair&gt;</span></span></em>  --query 'KeyMaterial' --output text | out-file -encoding ascii -filepath adsshkey
aws secretsmanager create-secret --region <em><span style="font-family: courier;"><span style="color: #ff0000;">&lt;Region&gt;</span></span></em> --name "adsshkey" --description "my ssh key pair" --secret-string file://adsshkey
</code></pre> 
</div> 
<blockquote>
 <p><strong>Note</strong>: Don’t change the name of the secret, as other scripts in the solution reference it. The worker EC2 instance will fetch the SSH key using <a href="https://docs.aws.amazon.com/secretsmanager/latest/apireference/API_GetSecretValue.html" rel="noopener noreferrer" target="_blank">GetSecretValue</a> API to SSH or RDP into other EC2 instances during domain join process.</p>
</blockquote> 
<h2>Deploy the solution</h2> 
<p>With the prerequisites in place, your next step is to download or clone the <a href="https://github.com/aws-samples/secure-ad-creds-join-unjoin-domain" rel="noopener noreferrer" target="_blank">GitHub</a> repo and store the files on your local machine. Go to the location where you cloned or downloaded the repo and review the contents of the <span style="font-family: courier;">config/OS_User_Mapping.json</span> file to validate the instance user name and operating system mapping. Update the file if you’re using a user name other than the one used to log in to the EC2 instances. The default user name used in this solution is <em>ec2-user</em> for Linux instances and <em>Administrator</em> for Windows.</p> 
<p>The solution requires installation of some software on the worker EC2 instance. Because the EC2 instance doesn’t have internet access, you must download the latest Windows 64-bit version of the following software to your local machine and upload it into the solution deployment S3 bucket in subsequent steps.</p> 
<ul> 
 <li><a href="https://stedolan.github.io/jq/download/" rel="noopener noreferrer" target="_blank">jq</a></li> 
 <li><a href="https://git-scm.com/download/win" rel="noopener noreferrer" target="_blank">Git client</a></li> 
 <li><a href="https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2-windows.html#cliv2-windows-install" rel="noopener noreferrer" target="_blank">AWS CLI version 2</a></li> 
</ul> 
<blockquote>
 <p><strong>Note</strong>: This step isn’t required if your EC2 instances have internet access.</p>
</blockquote> 
<p>Once done, use the following steps to deploy the solution in your AWS account:</p> 
<h3>Steps to deploy the solution:</h3> 
<ol> 
 <li>Create a private <a href="https://aws.amazon.com/s3/" rel="noopener noreferrer" target="_blank">Amazon Simple Storage Service (Amazon S3)</a> bucket using <a href="https://docs.aws.amazon.com/AmazonS3/latest/gsg/CreatingABucket.html" rel="noopener noreferrer" target="_blank">this documentation</a> to store the Lambda functions and the domain join and unjoin scripts.</li> 
 <li>Once created, enable versioning on this bucket using the following <a href="https://docs.aws.amazon.com/AmazonS3/latest/user-guide/enable-versioning.html" rel="noopener noreferrer" target="_blank">documentation</a>. Versioning lets you keep multiple versions of your objects in one bucket and helps you easily retrieve and restore previous versions of your scripts.</li> 
 <li>Upload the software you downloaded to the S3 bucket. This is only required if your instance doesn’t have internet access.</li> 
 <li>Upload the cloned or downloaded GitHub repo files to the S3 bucket.</li> 
 <li>Go to the S3 bucket and select the template name <span style="font-family: courier;"><span style="font-weight: bold;">secret-active-dir-solution.json</span></span>, and copy the object URL.</li> 
 <li>Open the CloudFormation console. Choose the appropriate AWS Region, and then choose <strong>Create Stack</strong>. Select <strong>With new resources</strong>.</li> 
 <li>Select <strong>Amazon S3 URL</strong> as the template source, paste the object URL that you copied in Step 5, and then choose <strong>Next</strong>.</li> 
 <li>On the <strong>Specify stack details</strong> page, enter a name for the stack and provide the following input parameters. You can modify the default values to customize the solution for your environment. 
  <ul> 
   <li><strong>ADUSECASE</strong> – From the dropdown menu, select your required use case. There is no default value.</li> 
   <li><strong>AdminUserId</strong> – The canonical user ID of the IAM user who manages the Active Directory credentials stored in Secrets Manager. To learn how to find the canonical user ID for your IAM user, scroll down to <em>Finding the canonical user ID for your AWS account</em> in <a href="https://docs.aws.amazon.com/general/latest/gr/acct-identifiers.html" rel="noopener noreferrer" target="_blank">AWS account identifiers</a>.</li> 
   <li><strong>DenyPolicyName</strong> – The name of the IAM policy that restricts access to the worker EC2 instance and the IAM role used by the worker to fetch credentials from Secrets Manager. You can keep the default value or provide another name.</li> 
   <li><strong>InstanceType</strong> – Instance type to be used when launching the worker EC2 instance. You can keep the default value or use another instance type if necessary.</li> 
   <li><strong>Placeholder</strong> – This is a dummy parameter that’s used as a placeholder in IAM policies for the EC2 instance ID. Keep the default value.</li> 
   <li><strong>S3Bucket</strong> – The name of the S3 bucket that you created in the first step of the solution deployment. Replace the default value with your S3 bucket name.</li> 
   <li><strong>S3prefix</strong> – Amazon S3 object key where the source scripts are stored. Leave the default value as long as the cloned GitHub directory structure hasn’t been changed.</li> 
   <li><strong>SSHKeyRequired</strong> – Select <strong>true</strong> or <strong>false</strong> based on whether an SSH key pair is required to RDP into the EC2 worker instance. If you select <strong>false</strong>, the worker EC2 instance will not have an SSH key pair.</li> 
   <li><strong>SecurityGroupId</strong> – Security group IDs to be associated with the worker instance to control traffic to and from the instance.</li> 
   <li><strong>Subnet</strong> – Select the VPC subnet where you want to launch the worker EC2 instance.</li> 
   <li><strong>VPC</strong> – Select the VPC where you want to launch the worker EC2 instance. Use the VPC where you have created the AWS Managed Microsoft AD.</li> 
   <li><strong>WorkerSSHKeyName</strong> – An existing SSH key pair name that can be used to get the password for RDP access into the EC2 worker instance. This isn’t mandatory if you’re using user name and password based login or <a href="https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager.html" rel="noopener noreferrer" target="_blank">AWS Systems Manager Session Manager</a>. This is required only if you have selected <strong>true</strong> for the <strong>SSHKeyRequired</strong> parameter.</li> 
  </ul> <p></p>
  <div class="wp-caption aligncenter" id="attachment_18848" style="width: 626px;">
   <img alt="Figure 2: Defining the stack name and input parameters for the CloudFormation stack" class="size-full wp-image-18848" height="892" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2021/02/15/Secure-automated-domain-EC2-2.png" width="616" />
   <p class="wp-caption-text" id="caption-attachment-18848">Figure 2: Defining the stack name and input parameters for the CloudFormation stack</p>
  </div></li> 
 <li>Enter values for all of the input parameters, and then choose <strong>Next</strong>.</li> 
 <li>On the <strong>Options</strong> page, keep the default values and then choose <strong>Next</strong>.</li> 
 <li>On the <strong>Review</strong> page, confirm the details, acknowledge that CloudFormation might create IAM resources with custom names, and choose <strong>Create Stack</strong>.</li> 
 <li>Once the stack creation is marked as <span style="font-family: courier;">CREATE_COMPLETE</span>, the following resources are created: 
  <ul> 
   <li>An EC2 instance that acts as a worker and runs Active Directory join scripts on the remote EC2 instances. It also unjoins instances from the domain upon instance termination.</li> 
   <li>A <em>secret</em> with a default Active Directory domain name, user name, and a dummy password. The name of the default secret is <em>myadcredV1</em>.</li> 
   <li>A Secrets Manager resource-based policy to deny all access to the secret except to the intended IAM users and roles.</li> 
   <li>An EC2 IAM profile and IAM role to be used only by the worker EC2 instance.</li> 
   <li>A managed IAM policy called <span style="font-family: courier;">DENYPOLICY</span> that can be assigned to an IAM user, group, or role to restrict access to the solution resources such as the worker EC2 instance.</li> 
   <li>A <a href="https://docs.aws.amazon.com/AmazonCloudWatch/latest/events/WhatIsCloudWatchEvents.html" rel="noopener noreferrer" target="_blank">CloudWatch Events</a> rule to detect <span style="font-family: courier;">running</span> and <span style="font-family: courier;">terminated</span> states for EC2 instances and trigger a <a href="https://docs.aws.amazon.com/lambda/latest/dg/welcome.html" rel="noopener noreferrer" target="_blank">Lambda function</a> that posts instance details to an <a href="https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/welcome.html" rel="noopener noreferrer" target="_blank">SQS queue</a>.</li> 
   <li>A Lambda function that reads instance tags and writes to an SQS queue based on the instance tag value, which can be <span style="font-family: courier;">true</span> or <span style="font-family: courier;">false</span>.</li> 
   <li>An <a href="https://aws.amazon.com/sqs/" rel="noopener noreferrer" target="_blank">SQS</a> queue for storing the EC2 instance state—<span style="font-family: courier;">running</span> or <span style="font-family: courier;">terminated</span>.</li> 
   <li>A <a href="https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-dead-letter-queues.html" rel="noopener noreferrer" target="_blank">dead-letter queue</a> for storing unprocessed messages.</li> 
   <li>An S3 bucket policy to restrict access to the source S3 bucket from unauthorized users or roles.</li> 
   <li>A CloudWatch log group to stream the logs of the worker EC2 instance.</li> 
  </ul> </li> 
</ol> 
<h2>Test the solution</h2> 
<p>Now that the solution is deployed, you can test it to check if it’s working as expected. Before you test the solution, you must navigate to the secret created in Secrets Manager by CloudFormation and update the Active Directory credentials—domain name, user name, and password.</p> 
<h3>To test the solution</h3> 
<ol> 
 <li>In the CloudFormation console, choose S<strong>ervices</strong>, and then <strong>CloudFormation</strong>. Select your stack name. On the stack <strong>Outputs</strong> tab, look for the <strong>ADSecret</strong> entry.</li> 
 <li>Choose the <strong>ADSecret</strong> link to go to the configuration for the secret in the Secrets Manager console. Scroll down to the section titled <strong>Secret value</strong>, and then choose <strong>Retrieve secret value</strong> to display the default <strong>Secret Key</strong> and <strong>Secret Value</strong> as shown in Figure 3. <p></p>
  <div class="wp-caption aligncenter" id="attachment_18851" style="width: 651px;">
   <img alt="Figure 3: Retrieve value in Secrets Manager" class="size-full wp-image-18851" height="366" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2021/02/15/Secure-automated-domain-EC2-3.png" width="641" />
   <p class="wp-caption-text" id="caption-attachment-18851">Figure 3: Retrieve value in Secrets Manager</p>
  </div></li> 
 <li>Choose the <strong>Edit</strong> button and update the default dummy credentials with your Active Directory domain credentials.(Optional) <strong>Directory_ou</strong> is used to store the organizational unit (OU) and directory components (DC) for the directory; for example, <span style="font-family: courier;">OU=test,DC=example,DC=com</span>.</li> 
</ol> 
<blockquote>
 <p><strong>Note</strong>: <strong>instance_password</strong> is an optional secret key and is used only when you’re using user name and password based login to access the EC2 instances in your account.</p>
</blockquote> 
<p>Now that the secret is updated with the correct credentials, you can launch a test EC2 instance and determine if the instance has successfully joined the Active Directory domain.</p> 
<h3>Create an Amazon Machine Image</h3> 
<blockquote>
 <p><strong>Note</strong>: This is only required for Linux-based operating systems other than <a href="https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/EC2_GetStarted.html" rel="noopener noreferrer" target="_blank">Amazon Linux</a>. You can skip these steps if your instances have internet access.</p>
</blockquote> 
<p>As your VPC doesn’t have internet access, for Linux-based systems other than Amazon Linux 1 or Amazon Linux 2, the required packages must be available on the instances that need to join the Active Directory domain. For that, you must create a custom Amazon Machine Image (AMI) from an EC2 instance with the required packages. If you already have a process to build your own AMIs, you can add these packages as part of that existing process.</p> 
<h4>To install the package into your AMI</h4> 
<ol> 
 <li>Create a new EC2 Linux instance for the required operating system in a public subnet or a private subnet with access to the internet via a NAT gateway.</li> 
 <li>Connect to the instance using any SSH client.</li> 
 <li>Install the required software by running the following command that is appropriate for the operating system: 
  <ul> 
   <li>For CentOS: 
    <div class="hide-language"> 
     <pre><code class="lang-text">yum -y install realmd adcli oddjob-mkhomedir oddjob samba-winbind-clients samba-winbind samba-common-tools samba-winbind-krb5-locator krb5-workstation unzip
</code></pre> 
    </div> </li> 
   <li>For RHEL: 
    <div class="hide-language"> 
     <pre><code class="lang-text">yum -y  install realmd adcli oddjob-mkhomedir oddjob samba-winbind-clients samba-winbind samba-common-tools samba-winbind-krb5-locator krb5-workstation python3 vim unzip
</code></pre> 
    </div> </li> 
   <li>For Ubuntu: 
    <div class="hide-language"> 
     <pre><code class="lang-text">apt-get -yq install realmd adcli winbind samba libnss-winbind libpam-winbind libpam-krb5 krb5-config krb5-locales krb5-user packagekit  ntp unzip python
</code></pre> 
    </div> </li> 
   <li>For SUSE: 
    <div class="hide-language"> 
     <pre><code class="lang-text">sudo zypper -n install realmd adcli sssd sssd-tools sssd-ad samba-client krb5-client samba-winbind krb5-client python
</code></pre> 
    </div> </li> 
   <li>For Debian: 
    <div class="hide-language"> 
     <pre><code class="lang-text">apt-get -yq install realmd adcli winbind samba libnss-winbind libpam-winbind libpam-krb5 krb5-config krb5-locales krb5-user packagekit  ntp unzip
</code></pre> 
    </div> </li> 
  </ul> </li> 
 <li>Follow <a href="https://docs.aws.amazon.com/directoryservice/latest/admin-guide/join_linux_instance.html" rel="noopener noreferrer" target="_blank">Manually join a Linux instance</a> to install the AWS CLI on Linux.</li> 
 <li>Create a new AMI based on this instance by following the instructions in <a href="https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/creating-an-ami-ebs.html#how-to-create-ebs-ami" rel="noopener noreferrer" target="_blank">Create a Linux AMI from an instance</a>.</li> 
</ol> 
<p>You now have a new AMI that can be used in the next steps and in future to launch similar instances.</p> 
<p>For Amazon Linux-based EC2 instances, the solution will use the mechanism described in <a href="https://aws.amazon.com/premiumsupport/knowledge-center/ec2-al1-al2-update-yum-without-internet/" rel="noopener noreferrer" target="_blank">How can I update yum or install packages without internet access on my EC2 instances</a> to install the required packages and you don’t need to create a custom AMI. No additional packages are required if you are using Windows-based EC2 instances.</p> 
<h3>To launch a test EC2 instance</h3> 
<ol> 
 <li>Navigate to the Amazon EC2 console and <a href="https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/LaunchingAndUsingInstances.html" rel="noopener noreferrer" target="_blank">launch an Amazon Linux or Windows EC2 instance</a> in the same Region and VPC that you used when creating the CloudFormation stack. For any other operating system, make sure you are using the custom AMI created before.</li> 
 <li>In the <strong>Add Tags</strong> section, add a tag named <span style="font-family: courier;">JoinAD</span> and set the value as <span style="font-family: courier;">true</span>. Add another tag named <span style="font-family: courier;">Operating_System</span> and set the appropriate operating system value from: 
  <ul> 
   <li><span style="font-family: courier;">AMAZON_LINUX</span></li> 
   <li><span style="font-family: courier;">FEDORA</span></li> 
   <li><span style="font-family: courier;">RHEL</span></li> 
   <li><span style="font-family: courier;">CENTOS</span></li> 
   <li><span style="font-family: courier;">UBUNTU</span></li> 
   <li><span style="font-family: courier;">DEBIAN</span></li> 
   <li><span style="font-family: courier;">SUSE</span></li> 
   <li><span style="font-family: courier;">WINDOWS</span></li> 
  </ul> </li> 
 <li>Make sure that the security group associated with this instance is set to allow all inbound traffic from the security group of the worker EC2 instance.</li> 
 <li>Use the SSH key pair name from the prerequisites (Step 6) when launching the instance.</li> 
 <li>Wait for the instance to launch and join the Active Directory domain. You can now navigate to the CloudWatch log group named <span style="font-family: courier;">/ad-domain-join-solution/</span> created by the CloudFormation stack to determine if the instance has joined the domain or not. On successful join, you can connect to the instance using a RDP or SSH client and entering your login credentials.</li> 
 <li>To test the domain unjoin workflow, you can terminate the EC2 instance launched in Step 1 and log in to the Active Directory tools instance to validate that the Active Directory computer object that represents the instance is deleted.</li> 
</ol> 
<h2>Solution review</h2> 
<p>Let’s review the details of the solution components and what happens during the domain join and unjoin process:</p> 
<h4>1) The worker EC2 instance:</h4> 
<p>The worker EC2 instance used in this solution is a Windows instance with all configurations required to add and remove machines to and from an Active Directory domain. It can also be used as an <a href="https://docs.aws.amazon.com/directoryservice/latest/admin-guide/ms_ad_install_ad_tools.html" rel="noopener noreferrer" target="_blank">Active Directory administration tools</a> instance. This instance is continuously running a bash script that is polling the SQS queue for new messages. Upon arrival of a new message, the script performs the following tasks:</p> 
<ol type="a"> 
 <li>Check if the instance is in running or terminated state to determine if it needs to be added or removed from the Active Directory domain.</li> 
 <li>If the message is from a newly launched EC2 instance, then this means that this instance needs to join the Active Directory domain.</li> 
 <li>The script identifies the instance operating system and runs the appropriate PowerShell or bash script on the remote EC2.</li> 
 <li>Similarly, if the instance is in terminated state, then the worker will run the domain unjoin command locally to remove the computer object from the Active Directory domain.</li> 
 <li>If the worker fails to process a message in the SQS queue, it sends the unprocessed message to a backup queue for debugging.</li> 
 <li>The worker writes logs related to the success or failure of the domain join to a CloudWatch log group. Use <span style="font-family: courier;">/ad-domain-join-solution</span> to filter for all other logs created by the worker instance in CloudWatch.</li> 
</ol> 
<h4>2) The worker bash script running on the instance:</h4> 
<p>This script polls the SQS queue every 5 seconds for new messages and is responsible for following activities:</p> 
<ul> 
 <li>Fetching Active Directory join credentials (user name and password) from Secrets Manager.</li> 
 <li>If the remote EC2 instance is running Windows, running the <span style="font-family: courier;">Invoke-Command</span> PowerShell cmdlet on the instance to perform the Active Directory join operation.</li> 
 <li>If the remote EC2 instance is running Linux, running <span style="font-family: courier;">realm join</span> command on the instance to perform the Active Directory join operation.</li> 
 <li>Running the <span style="font-family: courier;">Remove-ADComputer</span> command to remove the computer object from the Active Directory domain for terminated EC2 instances.</li> 
 <li>Storing domain-joined EC2 instance details—computer name and IP address—in an <a href="https://aws.amazon.com/dynamodb/" rel="noopener noreferrer" target="_blank">Amazon DynamoDB</a> table. These details are used to check if an instance is already part of the domain and when removing the instance from the Active Directory domain.</li> 
</ul> 
<h2>More information</h2> 
<p>Now that you have tested the solution, here are some additional points to be noted:</p> 
<ul> 
 <li>The Active Directory join and unjoin scripts provided with this solution can be replaced with your existing custom scripts.</li> 
 <li>To update the scripts on the worker instance, you must upload the modified scripts to the S3 bucket and the changes will automatically synchronize on the instance.</li> 
 <li>This solution works with single account, Region, and VPC combination. It can be modified to use across multiple Regions and VPC combinations.</li> 
 <li>For VPCs in a different account or Region, you must <a href="https://docs.aws.amazon.com/directoryservice/latest/admin-guide/ms_ad_tutorial_directory_sharing.html" rel="noopener noreferrer" target="_blank">share your AWS Managed Microsoft AD</a> with another AWS account when the networking prerequisites have been completed.</li> 
 <li>The instance user name and operating system mapping used in the solution is based on the <a href="https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/connection-prereqs.html#connection-prereqs-get-info-about-instance" rel="noopener noreferrer" target="_blank">default user name used by AWS</a>.</li> 
 <li>You can use AWS Systems Manager with VPC endpoints to log in to EC2 instances that don’t have internet access.</li> 
</ul> 
<p>The solution is protecting your Active Directory credentials and is making sure that:</p> 
<ul> 
 <li>Active Directory credentials can be accessed only from the worker EC2 instance.</li> 
 <li>The <a href="http://aws.amazon.com/iam" rel="noopener noreferrer" target="_blank">IAM</a> role used by the worker EC2 instance to fetch the secret cannot be assumed by other IAM entities.</li> 
 <li>Only authorized users can read the credentials from the Secrets Manager console, through <a href="http://aws.amazon.com/cli" rel="noopener noreferrer" target="_blank">AWS CLI</a>, or by using any other <a href="https://aws.amazon.com/tools/" rel="noopener noreferrer" target="_blank">AWS Tool</a>—such as an AWS SDK.</li> 
</ul> 
<p>The focus of this solution is to demonstrate a method you can use to secure Active Directory credentials and automate the process of EC2 instances joining and unjoining from an Active Directory domain.</p> 
<ul> 
 <li>You can associate the IAM policy named <span style="font-family: courier;">DENYPOLICY</span> with any IAM group or user in the account to block that user or group from accessing or modifying the worker EC2 instance and the IAM role used by the worker.</li> 
 <li>If your account belongs to an organization, you can use an organization-level <a href="https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_policies_scps.html" rel="noopener noreferrer" target="_blank">service control policy</a> instead of an IAM-managed policy—such as <span style="font-family: courier;">DENYPOLICY</span>—to protect the underlying resources from unauthorized users.</li> 
</ul> 
<h2>Conclusion</h2> 
<p>In this blog post, you learned how to deploy an automated and secure solution through CloudFormation to help secure the Active Directory credentials and also manage adding and removing Amazon EC2 instances to and from an Active Directory domain. When using this solution, you incur Amazon EC2 charges along with charges associated with <a href="https://aws.amazon.com/secrets-manager/pricing/" rel="noopener noreferrer" target="_blank">Secrets Manager pricing</a> and <a href="https://aws.amazon.com/privatelink/pricing/" rel="noopener noreferrer" target="_blank">AWS PrivateLink</a>.</p> 
<p>You can use the following references to help diagnose or troubleshoot common errors during the domain join or unjoin process.</p> 
<ul> 
 <li><a href="https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_remote_troubleshooting?view=powershell-7" rel="noopener noreferrer" target="_blank">Troubleshoot remote operations in PowerShell</a></li> 
 <li><a href="https://docs.aws.amazon.com/directoryservice/latest/admin-guide/ms_ad_troubleshooting.html" rel="noopener noreferrer" target="_blank">Troubleshooting AWS Managed Microsoft AD</a></li> 
 <li><a href="https://support.microsoft.com/en-in/help/4341920/troubleshoot-errors-when-you-join-windows-based-computers-to-domain" rel="noopener noreferrer" target="_blank">How to troubleshoot errors that occur when you join Windows-based computers to a domain</a></li> 
 <li><a href="https://docs.aws.amazon.com/directoryservice/latest/admin-guide/ms_ad_troubleshooting_join_linux.html" rel="noopener noreferrer" target="_blank">Troubleshooting Linux domain join errors</a></li> 
</ul> 
<p>If you have feedback about this post, submit comments in the <strong>Comments</strong> section below.</p> 
<p><strong>Want more AWS Security how-to content, news, and feature announcements? Follow us on <a href="https://twitter.com/AWSsecurityinfo" rel="noopener noreferrer" target="_blank" title="Twitter">Twitter</a>.</strong></p> 
<footer> 
 <div class="blog-author-box"> 
  <div class="blog-author-image">
   <img alt="Author" class="aligncenter size-full wp-image-18859" height="160" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2021/02/15/Rakesh-Singh-Author.jpg" width="120" />
  </div> 
  <h3 class="lb-h4">Rakesh Singh</h3> 
  <p>Rakesh is a Technical Account Manager with AWS. He loves automation and enjoys working directly with customers to solve complex technical issues and provide architectural guidance. Outside of work, he enjoys playing soccer, singing karaoke, and watching thriller movies.</p> 
 </div> 
</footer>
