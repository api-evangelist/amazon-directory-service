---
title: "Manage your AWS Directory Service credentials using AWS Secrets Manager"
url: "https://aws.amazon.com/blogs/security/manage-your-aws-directory-service-credentials-using-aws-secrets-manager/"
date: "Tue, 28 Sep 2021 16:53:15 +0000"
author: "Ashwin Bhargava"
feed_url: "https://aws.amazon.com/blogs/security/tag/aws-directory-service/feed/"
---
<p><a href="https://aws.amazon.com/secrets-manager" rel="noopener noreferrer" target="_blank">AWS Secrets Manager</a> helps you protect the secrets that are needed to access your applications, services, and IT resources. With this service, you can rotate, manage, and retrieve database credentials, API keys, OAuth tokens, and other secrets throughout their lifecycle. The secret value rotation feature has built-in integration for services like <a href="https://aws.amazon.com/rds/" rel="noopener noreferrer" target="_blank">Amazon Relational Database Service (Amazon RDS)</a> , whose credentials can be rotated. The same integration functionality can also be extended to other types of secrets, including API keys and OAuth tokens, with the help of <a href="https://aws.amazon.com/lambda/" rel="noopener noreferrer" target="_blank">AWS Lambda</a> functions.</p> 
<p>This blog post provides details on how <a href="https://aws.amazon.com/secrets-manager" rel="noopener noreferrer" target="_blank">Secrets Manager</a> can be used to store and rotate the admin password of <a href="https://aws.amazon.com/directoryservice/" rel="noopener noreferrer" target="_blank">AWS Directory Service</a> at a specified frequency. Customers who use the directory services in AWS can deploy the solution in this blog post to minimize the effort spent by their operations team to manually rotate the password (which is one of the best practices of password management). These customers can also benefit by using the secure API access of Secrets Manager to allow access by applications that are using Active Directory–specific accounts. A good example is having an application to reset passwords for AD users and can be done using the API access.</p> 
<h2>Solution overview</h2> 
<p>When you configure AWS Directory Service, one of the inputs the service expects is the password for the admin user (administrator).&nbsp;By using an <a href="https://aws.amazon.com/lambda/" rel="noopener noreferrer" target="_blank">AWS Lambda</a> function and <a href="https://aws.amazon.com/secrets-manager" rel="noopener noreferrer" target="_blank">Secrets Manager</a>, you can store the password and rotate it periodically.</p> 
<p>Figure 1 shows the architecture diagram for this solution.<br /> &nbsp;<br /> </p>
<div class="wp-caption aligncenter" id="attachment_22342" style="width: 506px;">
 <img alt="Figure 1: Architecture diagram" class="size-full wp-image-22342" height="381" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2021/09/23/Manage-AWS-Directory-Service-credentials-1.jpg" width="496" />
 <p class="wp-caption-text" id="caption-attachment-22342">Figure 1: Architecture diagram</p>
</div>
<p></p> 
<p>The workflow is as follows:</p> 
<ol> 
 <li>During initial setup (which can be performed either manually or through a CloudFormation template), the password of the admin user is stored as a secret in Secrets Manager. The secret is in the JSON format and contains three fields: Directory ID, UserName, and Password. The secret is encrypted using KMS Key to provide an added layer of security.</li> 
 <li>This secret is attached to a Lambda function that controls rotation.</li> 
 <li>This rotation Lambda function generates a new password, updates Active Directory, and then updates the secret. The function can be invoked on as-needed basis or at a desired interval. The CFN template we provide in this post schedules the rotation at a 30-day interval.</li> 
 <li>Applications can securely fetch the new secret value from Secrets Manager.</li> 
</ol> 
<h2>Prerequisites and assumptions</h2> 
<p>To implement this solution, you need an AWS account to test the solution and access AWS services.</p> 
<p>Also be aware of the following:</p> 
<ol> 
 <li>In this solution, you will configure all the (supported) services in the same virtual private cloud (VPC) to simplify networking considerations.</li> 
 <li>The predefined admin user name for Simple Active Directory is <span style="font-family: courier;">Administrator</span>.</li> 
 <li>The predefined password is a random 12-character string.</li> 
</ol> 
<blockquote>
 <p><strong>Important</strong>: The&nbsp;<a href="http://aws.amazon.com/cloudformation" rel="noopener noreferrer" target="_blank">AWS CloudFormation</a>&nbsp;template that we provide deploys a Simple Active Directory. This is for testing and demonstration purposes; you can modify or reuse the solution for other types of Active Directory solutions.</p>
</blockquote> 
<h2>Deploy the solution</h2> 
<p>To deploy the solution, you first provision the baseline networking and other resources by using a CloudFormation stack.</p> 
<p>The resource provisioning in this step creates these resources:</p> 
<ul> 
 <li>An <a href="https://aws.amazon.com/vpc" rel="noopener noreferrer" target="_blank">Amazon Virtual Private Cloud (Amazon VPC)</a> with two private subnets</li> 
 <li>AWS Directory Service installed and configured in the VPC</li> 
 <li>A Secrets Manager secret with rotation enabled</li> 
 <li>A Lambda function inside the VPC</li> 
 <li>These <a href="http://aws.amazon.com/iam" rel="noopener noreferrer" target="_blank">AWS Identity and Access Management (IAM)</a> roles and permissions: 
  <ul> 
   <li>Secrets Manager has permission to invoke Lambda functions</li> 
   <li>The Lambda function has permission to update the secret in Secrets Manager</li> 
   <li>The Lambda function has permission to update the password for Directory Service</li> 
  </ul> </li> 
</ul> 
<h3>To deploy the solution by using the CloudFormation template</h3> 
<ol> 
 <li>You can use&nbsp;<a href="https://aws-security-blog-content.s3.amazonaws.com/public/sample/775-Store-rotate-Directory-Service-credentials/master-template.yml" rel="noopener noreferrer" target="_blank">this downloadable template</a> to set up the resources. To launch directly through the console, choose the following&nbsp;<strong>Launch Stack</strong>&nbsp;button, which creates the stack in the us-east-1 AWS Region.<br /> <a href="https://console.aws.amazon.com/cloudformation/home?region=us-east-1#/stacks/new?stackName=aws-ds-creds-manager&amp;templateURL=https://aws-security-blog-content.s3.amazonaws.com/public/sample/775-Store-rotate-Directory-Service-credentials/master-template.yml" rel="noopener noreferrer" target="_blank"> <img alt="Select the Launch Stack button to launch the template" class="aligncenter size-full wp-image-15619" height="63" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2020/08/31/Launch-Stack-Button-2020.png" width="232" /></a> </li> 
 <li>Choose <strong>Next</strong> to go to the <strong>Specify stack details</strong> page.</li> 
 <li>The bucket hosting the Lambda function code is predefined for ease of implementation, but you can edit the bucket name if necessary. Specify any other template details as needed, and then choose&nbsp;<strong>Next</strong>.</li> 
 <li>(Optional) On the&nbsp;<strong>Configure Stack Options</strong> page, enter any tags, and then choose <strong>Next</strong>.</li> 
 <li>On the&nbsp;<strong>Review</strong>&nbsp;page, select the check box for&nbsp;<strong>I acknowledge that AWS CloudFormation might create IAM resources with custom names</strong>, and choose&nbsp;<strong>Create stack</strong>.</li> 
</ol> 
<p>It takes approximately 20–25 minutes for the provisioning to complete. When the stack status shows <strong>Create Complete</strong>, review the outputs that were created by navigating to the <strong>Outputs</strong> tab, as shown in Figure 2.<br /> &nbsp;<br /> </p>
<div class="wp-caption aligncenter" id="attachment_22343" style="width: 2146px;">
 <img alt="Figure 2: Outputs created by the CloudFormation template" class="size-full wp-image-22343" height="832" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2021/09/23/Manage-AWS-Directory-Service-credentials-2.png" width="2136" />
 <p class="wp-caption-text" id="caption-attachment-22343">Figure 2: Outputs created by the CloudFormation template</p>
</div>
<p></p> 
<p>Now that the stack creation has completed successfully, you should validate the resources that were created.</p> 
<h3>To validate the resources</h3> 
<ol> 
 <li>Navigate to the AWS Directory Service console. You should see a new directory service that has the <span style="font-family: courier;">corp.com</span> directory set up.</li> 
 <li>Navigate to the AWS Secrets Manager console and review the secret that was created called <strong>DSAdminPswd</strong>. Choose the secret value, and then choose <strong>Retrieve secret value </strong>to reveal the secret values.<br /> &nbsp;<br /> 
  <div class="wp-caption aligncenter" id="attachment_22344" style="width: 829px;">
   <img alt="Figure 3: Checking the secret value in the Secrets Manager console" class="size-full wp-image-22344" height="275" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2021/09/23/Manage-AWS-Directory-Service-credentials-3.png" width="819" />
   <p class="wp-caption-text" id="caption-attachment-22344">Figure 3: Checking the secret value in the Secrets Manager console</p>
  </div> </li> 
 <li>As you might have noticed, the secret value changed from what was initially generated in the template. The Lambda function was invoked when it was attached to the secret, which caused the secret to rotate. To verify that the secret value changed, navigate to the Amazon CloudWatch console, and then navigate to <strong>Log groups</strong>.</li> 
 <li>In the search bar, type the Lambda function name <span style="font-family: courier;">dj-rotate-lambda</span> to filter on the log group name.<br /> &nbsp;<br /> 
  <div class="wp-caption aligncenter" id="attachment_22345" style="width: 2798px;">
   <img alt="Figure 4: CloudWatch log groups" class="size-full wp-image-22345" height="494" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2021/09/23/Manage-AWS-Directory-Service-credentials-4.png" width="2788" />
   <p class="wp-caption-text" id="caption-attachment-22345">Figure 4: CloudWatch log groups</p>
  </div> </li> 
 <li>Choose the log group <strong>/aws/lambda/dj-rotate-lambda</strong> to open the detailed log streams.</li> 
 <li>Look at the <strong>Log streams</strong> and open the recent log stream to view the series of rotation events.<br /> &nbsp;<br /> 
  <div class="wp-caption aligncenter" id="attachment_22346" style="width: 1447px;">
   <img alt="Figure 5: The log data for a complete rotation" class="size-full wp-image-22346" height="650" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2021/09/23/Manage-AWS-Directory-Service-credentials-5.png" width="1437" />
   <p class="wp-caption-text" id="caption-attachment-22346">Figure 5: The log data for a complete rotation</p>
  </div><p></p> <p> You should see that each of the four stages of rotation (create, set, test, and finish) are called in the right sequence. A Success message in the <span style="font-family: courier;">finishSecret</span> stage confirms the successful rotation of the secret value. </p></li> 
</ol> 
<p>The next step is to rotate the secret manually or set a policy for rotation.</p> 
<h3>To rotate the secret</h3> 
<p>The CloudFormation automation has set the rotation configuration to rotate the secret every 30 days. You can alternatively initiate another rotation by choosing <strong>Rotate secret immediately</strong>, as shown in Figure 6. You will observe the log stream (in CloudWatch Logs) changing, followed by the new secret value.<br /> &nbsp;<br /> </p>
<div class="wp-caption aligncenter" id="attachment_22347" style="width: 2370px;">
 <img alt="Figure 6: Manual rotation of the secret" class="size-full wp-image-22347" height="560" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2021/09/23/Manage-AWS-Directory-Service-credentials-6.png" width="2360" />
 <p class="wp-caption-text" id="caption-attachment-22347">Figure 6: Manual rotation of the secret</p>
</div>
<p></p> 
<p>You can also edit the rotation configuration by choosing <strong>Edit rotation </strong>and configuring the rotation policy that suits your organizational standards, as shown in Figure 7.<br /> &nbsp;<br /> </p>
<div class="wp-caption aligncenter" id="attachment_22348" style="width: 410px;">
 <img alt="Figure 7: Editing the rotation configuration" class="size-full wp-image-22348" height="306" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2021/09/23/Manage-AWS-Directory-Service-credentials-7.png" width="400" />
 <p class="wp-caption-text" id="caption-attachment-22348">Figure 7: Editing the rotation configuration</p>
</div>
<p></p> 
<h2>Code walkthrough</h2> 
<p>The rotation Lambda function works in four stages:</p> 
<ol> 
 <li><strong>CreateSecret</strong> – In this stage, the Lambda function creates a new password for the administrator user and sets up the staging label <span style="font-family: courier;">AWSPENDING</span> for the secret’s new value.</li> 
 <li><strong>SetSecret</strong> – In this stage, the Lambda function fetches the newly generated password by using the label <span style="font-family: courier;">AWSPENDING</span> and sets it as the password to the Active Directory administrator user.</li> 
 <li><strong>TestSecret</strong> – In this stage, the Lambda function verifies that the password is working by using the kinit command and the necessary dependent libraries of the Linux OS (the base OS for Lambda functions). If successful, the function continues to the next stage. In the case of failure, the catch block reverts the password of the Active Directory administrator user to the value in the <span style="font-family: courier;">AWSCURRENT</span> label.</li> 
 <li><strong>FinishSecret</strong> – This is the final stage, where the Lambda function moves the labels <span style="font-family: courier;">AWSCURRENT</span> from the current version of secret to the new version. And the same time, the old version of the secret is given <span style="font-family: courier;">AWSPREVIOUS</span> label.</li> 
</ol> 
<p>The Lambda function is written in Python 3.7 runtime and uses <a href="https://aws.amazon.com/sdk-for-python/" rel="noopener noreferrer" target="_blank">AWS SDK for Python (Boto3)</a> API calls for interacting with Secrets Manager and Directory Services.</p> 
<p>The directory ID and Secrets Manager endpoint are supplied as environment variables to the Lambda function, as shown in Figure 8. The secret ID is fetched from the event context.<br /> &nbsp;<br /> </p>
<div class="wp-caption aligncenter" id="attachment_22349" style="width: 2928px;">
 <img alt="Figure 8: Environment variables setup" class="size-full wp-image-22349" height="552" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2021/09/23/Manage-AWS-Directory-Service-credentials-8.png" width="2918" />
 <p class="wp-caption-text" id="caption-attachment-22349">Figure 8: Environment variables setup</p>
</div>
<p></p> 
<p>You can download the Lambda code that is used for the rotation logic and modify it to suit your organizational needs. For instance, the random password is configured to have a length of 12 characters, excluding special characters and punctuations, as shown in the following code snippet. You can modify this configuration as needed.</p> 
<div class="hide-language"> 
 <pre><code class="lang-text">newpasswd = service_client.get_random_password(PasswordLength=12,ExcludeCharacters='/@"\'\\',ExcludePunctuation=True)
</code></pre> 
</div> 
<p>As mentioned in the Prerequisites section, make sure that you do proper testing in development or test environments before proceeding to deploy the solution in production environments.</p> 
<h2>Cleanup</h2> 
<p>After you complete and test this solution, clean up the resources by deleting the AWS CloudFormation stack called <span style="font-family: courier;">aws-ds-creds-manager</span>. For more information on deleting the stacks, see <a href="https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/cfn-console-delete-stack.html" rel="noopener noreferrer" target="_blank">Deleting a stack on the AWS CloudFormation console</a>.</p> 
<h2>Conclusion</h2> 
<p>In this post, we demonstrated how to use the <a href="https://aws.amazon.com/secrets-manager" rel="noopener noreferrer" target="_blank">AWS Secrets Manager</a> service to store and rotate the <a href="http://aws.amazon.com/directoryservice" rel="noopener noreferrer" target="_blank">AWS Directory Service</a> Simple Active Directory admin password. You can also use this solution to rotate the <a href="https://aws.amazon.com/directoryservice/active-directory/" rel="noopener noreferrer" target="_blank">AWS Managed Microsoft AD</a> directory.</p> 
<p>There are many other <a href="https://docs.aws.amazon.com/code-samples/latest/catalog/code-catalog-lambda_functions-secretsmanager.html" rel="noopener noreferrer" target="_blank">code samples listed in the AWS Code Sample Catalog</a> that show how to rotate the passwords for other database services that are supported by this service.</p> 
<p>You can find additional rotation Lambda function examples in the <a href="https://github.com/aws-samples/aws-secrets-manager-rotation-lambdas" rel="noopener noreferrer" target="_blank">open source AWS library for Secrets Manager</a>.</p> 
<p>If you have feedback about this post, submit comments in the <strong>Comments</strong> section below. If you have questions about this post, start a new thread on the <a href="https://forums.aws.amazon.com/forum.jspa?forumID=296" rel="noopener noreferrer" target="_blank">AWS Secrets Manager forum</a> or <a href="https://console.aws.amazon.com/support/home" rel="noopener noreferrer" target="_blank" title="contact AWS Support">contact AWS Support</a>.</p> 
<p><strong>Want more AWS Security how-to content, news, and feature announcements? Follow us on <a href="https://twitter.com/AWSsecurityinfo" rel="noopener noreferrer" target="_blank" title="Twitter">Twitter</a>.</strong></p> 
<footer> 
 <div class="blog-author-box"> 
  <div class="blog-author-image"> 
   <img alt="Author" class="aligncenter size-full wp-image-22351" height="160" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2021/09/23/Ashwin-Bhargava-Author.jpg" width="120" /> 
  </div> 
  <h3 class="lb-h4">Ashwin Bhargava</h3> 
  <p>Ashwin is a DevOps Consultant at AWS working in Professional Services Canada. He is a DevOps expert and a security enthusiast with more than 13 years of development and consulting experience.</p> 
  <p></p>
 </div> 
</footer> 
<footer> 
 <div class="blog-author-box"> 
  <div class="blog-author-image"> 
   <img alt="Author" class="aligncenter size-full wp-image-22365" height="160" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2021/09/24/Satya-Vajrapu-Author.jpg" width="120" /> 
  </div> 
  <h3 class="lb-h4">Satya Vajrapu</h3> 
  <p>Satya is a Senior DevOps Consultant with AWS. He works with customers to help design, architect, and develop various practices and tools in the DevOps and cloud toolchain.</p> 
  <p></p>
 </div> 
</footer>
