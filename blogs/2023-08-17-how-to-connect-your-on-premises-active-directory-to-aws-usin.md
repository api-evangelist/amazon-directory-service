---
title: "How to Connect Your On-Premises Active Directory to AWS Using AD Connector"
url: "https://aws.amazon.com/blogs/security/how-to-connect-your-on-premises-active-directory-to-aws-using-ad-connector/"
date: "Thu, 17 Aug 2023 16:36:48 +0000"
author: "Jeremy Cowan"
feed_url: "https://aws.amazon.com/blogs/security/tag/aws-directory-service/feed/"
---
<blockquote>
 <p><strong>August 17, 2023:</strong> We updated the instructions and screenshots in this post to align with changes to the AWS Management Console.</p>
</blockquote> 
<blockquote>
 <p><strong>April 25, 2023:</strong> We’ve updated this blog post to include more security learning resources.</p>
</blockquote> 
<hr /> 
<p>AD Connector is designed to give you an easy way to establish a trusted relationship between your Active Directory and AWS. When AD Connector is configured, the trust allows you to:</p> 
<ul> 
 <li>Sign in to AWS applications such as Amazon WorkSpaces, Amazon WorkDocs, and Amazon WorkMail by using your Active Directory credentials.</li> 
 <li>Seamlessly join Windows instances to your Active Directory domain either through the Amazon EC2 launch wizard or programmatically through the EC2 Simple System Manager (SSM) API.</li> 
 <li>Provide federated sign-in to the AWS Management Console by mapping Active Directory identities to AWS Identity and Access Management (IAM) roles.</li> 
</ul> 
<p>AD Connector cannot be used with your custom applications, as it is only used for secure AWS integration for the three use-cases mentioned above. Custom applications relying on your on-premises Active Directory should communicate with your domain controllers directly or utilize <a href="https://aws.amazon.com/directoryservice/" rel="noopener" target="_blank">AWS Managed Microsoft AD</a> rather than integrating with AD Connector. To learn more about which AWS Directory Service solution works best for your organization, see the service <a href="https://docs.aws.amazon.com/directoryservice/latest/admin-guide/what_is.html" rel="noopener" target="_blank">documentation</a>.</p> 
<p>With AD Connector, you can streamline<a href="https://learnsecurity.amazon.com/en/index.html" rel="noopener" target="_blank"> identity management</a> by extending your user identities from Active Directory. It also enables you to reuse your existing Active Directory security policies such as password expiration, password history, and account lockout policies. Also, your users will no longer need to remember yet another user name and password combination. Since AD Connector doesn’t rely on complex directory synchronization technologies or Active Directory Federation Services (AD FS), you can forego the added cost and complexity of hosting a <a href="https://aws.amazon.com/iam/details/saml/" rel="noopener" target="_blank">SAML</a>-based <a href="https://aws.amazon.com/iam/details/manage-federation/" rel="noopener" target="_blank">federation </a>infrastructure. In sum, AD Connector helps foster a hybrid environment by allowing you to leverage your existing on-premises investments to control different facets of AWS.</p> 
<p>This blog post will show you how AD Connector works as well as walk through how to enable federated console access, assign users to roles, and seamlessly join an EC2 instance to an Active Directory domain.</p> 
<h2>AD Connector – Under the Hood</h2> 
<p>AD Connector is a dual Availability Zone proxy service that connects AWS apps to your on-premises directory. AD Connector forwards sign-in requests to your Active Directory domain controllers for authentication and provides the ability for applications to query the directory for data. When you configure AD Connector, you provide it with service account credentials that are securely stored by AWS. This account is used by AWS to enable seamless domain join, single sign-on (SSO), and AWS Applications (WorkSpaces, WorkDocs, and WorkMail) functionality. Given AD Connector’s role as a proxy, it does not store or cache user credentials. Rather, authentication, lookup, and management requests are handled by your Active Directory.</p> 
<p>In order to create an AD Connector, you must also provide a pair of DNS IP addresses during setup. These are used by AD Connector to retrieve Service (SRV) DNS records to locate the nearest domain controllers to route requests to. The AD connector proxy instances use an algorithm similar to the Active Directory domain controller locator process to decide which domain controllers to connect to for LDAP and Kerberos requests.</p> 
<p>For authentication to AWS applications and the AWS Management Console, you can configure an access URL from the AWS Directory Service console. This access URL is in the format of https://<span style="font-family: courier; color: #ff0000; font-style: italic;">&lt;alias&gt;</span>.awsapps.com and provides a publicly accessible sign-in page. You can visit https://<span style="font-family: courier; color: #ff0000; font-style: italic;">&lt;alias&gt;</span>.awsapps.com/workdocs to sign in to WorkDocs, and https://<span style="font-family: courier; color: #ff0000; font-style: italic;">&lt;alias&gt;</span>.awsapps.com/console to sign in to the AWS Management Console. The following image shows the sign-in page for the AWS Management Console.</p> 
<div class="wp-caption alignnone" id="attachment_30509" style="width: 386px;">
 <img alt="Figure 1: Login" class="size-full wp-image-30509" height="548" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2023/08/15/img1.png" style="border: 1px solid #bebebe;" width="376" />
 <p class="wp-caption-text" id="caption-attachment-30509">Figure 1: Login</p>
</div> 
<p>For added security you can enable multi-factor authentication (MFA) for AD Connector, but you’ll need to have an existing RADIUS infrastructure in your on-premises network set up to leverage this feature. See <a href="https://docs.aws.amazon.com/directoryservice/latest/admin-guide/prereq_connector.html" rel="noopener" target="_blank">AD Connector – Multi-factor Authentication Prerequisites</a> for more information about requirements and configuration. With MFA enabled with AD Connector, the sign-in page hosted at your access URL will prompt users for an MFA code in addition to their standard sign-in credentials.</p> 
<p>AD Connector comes in two sizes: small and large. A large AD Connector runs on more powerful compute resources and is more expensive than a small AD Connector. Depending on the volume of traffic to be proxied by AD Connector, you’ll want to select the appropriate size for your needs.</p> 
<div class="wp-caption alignnone" id="attachment_30510" style="width: 710px;">
 <img alt="Figure 2: Directory size" class="size-full wp-image-30510" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2023/08/15/img2.png" style="border: 1px solid #bebebe;" width="700" />
 <p class="wp-caption-text" id="caption-attachment-30510">Figure 2: Directory size</p>
</div> 
<p>AD Connector is highly available, meaning underlying hosts are deployed across multiple Availability Zones in the region you deploy. In the event of host-level failure, Directory Service will promptly replace failed hosts. Directory Service also applies performance and security updates automatically to AD Connector.</p> 
<p>The following diagram illustrates the authentication flow and network path when you enable AWS Management Console access:</p> 
<ol> 
 <li>A user opens the secure custom sign-in page and supplies their Active Directory user name and password.</li> 
 <li>The authentication request is sent over SSL to AD Connector.</li> 
 <li>AD Connector performs LDAP authentication to Active Directory.<br /> 
  <blockquote>
   <p><strong>Note:</strong> AD Connector locates the nearest domain controllers by querying the SRV DNS records for the domain.</p>
  </blockquote> </li> 
 <li>After the user has been authenticated, AD Connector calls the STS AssumeRole method to get temporary security credentials for that user. Using those temporary security credentials, AD Connector constructs a sign-in URL that users use to access the console.<br /> 
  <blockquote>
   <p><strong>Note:</strong> If a user is mapped to multiple roles, the user will be presented with a choice at sign-in as to which role they want to assume. The user session is valid for 1 hour.</p>
  </blockquote> 
  <div class="wp-caption alignnone" id="attachment_30511" style="width: 750px;">
   <img alt="Figure 3: Authentication flow and network path" class="size-full wp-image-30511" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2023/08/15/img3.png" style="border: 1px solid #bebebe;" width="740" />
   <p class="wp-caption-text" id="caption-attachment-30511">Figure 3: Authentication flow and network path</p>
  </div> </li> 
</ol> 
<p>Before getting started with configuring AD Connector for federated AWS Management Console access, be sure you’ve read and understand the <a href="https://docs.aws.amazon.com/directoryservice/latest/admin-guide/prereq_connector.html" rel="noopener" target="_blank">prerequisites for AD Connector</a>. For example, as shown in Figure 3 there must be a VPN or Direct Connect circuit in place between your VPC and your on-premises environment. Your domain also has to be running at Windows 2003 functional level or later. Also, various ports have to be opened between your VPC and your on-premises environment to allow AD Connector to communicate with your on-premises directory.</p> 
<h2>Configuring AD Connector for federated AWS Management Console access</h2> 
<h3>Enable console access</h3> 
<p>To allow users to sign in with their Active Directory credentials, you need to explicitly enable console access. You can do this by opening the Directory Service console and clicking the Directory ID name (Figure 4).</p> 
<p>This opens the <strong>Directory Details</strong> page, where you’ll find a dropdown menu on the <strong>Apps &amp; Services</strong> tab to enable the directory for AWS Management Console access.</p> 
<div class="wp-caption alignnone" id="attachment_30512" style="width: 790px;">
 <img alt="Figure 4: Directories" class="size-full wp-image-30512" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2023/08/15/img4-1.png" style="border: 1px solid #bebebe;" width="780" />
 <p class="wp-caption-text" id="caption-attachment-30512">Figure 4: Directories</p>
</div> 
<p>Choose the <strong>Application management</strong> tab as seen in Figure 5.</p> 
<div class="wp-caption alignnone" id="attachment_30514" style="width: 767px;">
 <img alt="Figure 5: Application Management" class="size-full wp-image-30514" height="89" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2023/08/15/img5-2.png" style="border: 1px solid #bebebe;" width="757" />
 <p class="wp-caption-text" id="caption-attachment-30514">Figure 5: Application Management</p>
</div> 
<p>Scroll down to <strong>AWS Management Console</strong> as shown in Figure 6, and choose <strong>Enable</strong> from the <strong>Actions</strong> dropdown list.</p> 
<div class="wp-caption alignnone" id="attachment_30515" style="width: 790px;">
 <img alt="Figure 6: Enable console access" class="size-full wp-image-30515" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2023/08/15/img6-1.png" style="border: 1px solid #bebebe;" width="780" />
 <p class="wp-caption-text" id="caption-attachment-30515">Figure 6: Enable console access</p>
</div> 
<p>After enabling console access, you’re ready to start configuring roles and associating Active Directory users and groups with those roles.</p> 
<p>Follow <a href="https://docs.aws.amazon.com/directoryservice/latest/admin-guide/create_role.html" rel="noopener" target="_blank">these steps</a> to create a new role. When you create a new role through the Directory Service console, AD Connector automatically adds a trust relationship to Directory Service. The following code example shows the IAM trust policy for the role, after a role is created.</p> 
<div class="hide-language"> 
 <pre><code class="lang-text">{
   "Version": "2012-10-17",
   "Statement": [
     {
       "Sid": "",
       "Effect": "Allow",
       "Principal": {
         "Service": "ds.amazonaws.com"
       },
       "Action": "sts:AssumeRole",
       "Condition": {
         "StringEquals": {
           "sts:externalid": "482242153642"
	  }
	}
     }
   ]
}</code></pre> 
</div> 
<h3>Assign users to roles</h3> 
<p>Now that AD Connector is configured and you’ve created a role, your next job is to assign users or groups to those IAM roles. Role mapping is what governs what resources a user has access to within AWS. To do this you’ll need to do the following steps:</p> 
<ol> 
 <li>Open the <a href="https://console.aws.amazon.com/directoryservice/" rel="noopener" target="_blank">Directory Service console</a> and navigate to the AWS Management Console section.</li> 
 <li>In the search bar, type the name of the role you just created.</li> 
 <li>Select the role that you just created by choosing the name under the <strong>IAM role</strong> field. </li> 
 <li>Choose <strong>Add</strong>, and enter the name to be added to find users or groups for this role.</li> 
 <li>Choose <strong>Add</strong>, and the user or group is now assigned to the role.</li> 
</ol> 
<p>When you’re finished, you should see the name of the user or group along with the corresponding ID for that object. It is also important to note that this list can be used to remove users or groups from the role. The next time the user signs in to the AWS Management Console from the custom sign-in page, they will be signed in under the <span style="font-family: courier;">EC2ReadOnly</span> security role.</p> 
<h2>Seamlessly join an instance to an Active Directory domain</h2> 
<p>Another advantage to using AD Connector is the ability to seamlessly join Windows (EC2) instances to your Active Directory domain. This allows you to join a Windows Server to the domain while the instance is being provisioned instead of using a script or doing it manually. This section of this blog post will explain the steps necessary to enable this feature in your environment and how the service works.</p> 
<h3>Step 1: Create a role</h3> 
<p>Until recently you had to manually create an IAM policy to allow an EC2 instance to access the SSM, an AWS service that allows you to configure Windows instances while they’re running and on first launch. Now, there’s a managed policy called <span style="font-family: courier;">AmazonEC2RoleforSSM</span> that you can use instead. The role you are about to create will be assigned to an EC2 instance when it’s provisioned, which will grant it permission to access the SSM service.</p> 
<p>To create the role:</p> 
<ol> 
 <li>Open the IAM console.</li> 
 <li>Click <strong>Roles</strong> in the navigation pane.</li> 
 <li>Click <strong>Create Role</strong>.</li> 
 <li>Type a name for your role in the <strong>Role Name</strong> field.</li> 
 <li>Under <strong>AWS Service Roles</strong>, select <strong>Amazon EC2</strong> and then click <strong>Select</strong>.</li> 
 <li>On the <strong>Attach Policy</strong> page, select <strong>AmazonEC2RoleforSSM</strong> and then click <strong>Next Step</strong>.</li> 
 <li>On the <strong>Review</strong> page, click <strong>Create Role</strong>.</li> 
</ol> 
<p>If you click the role you created, you’ll see a trust policy for EC2, which looks like the following code example.</p> 
<div class="hide-language"> 
 <pre><code class="lang-text">{
     "Version": "2012-10-17",
     "Statement": [
       {
         "Sid": "",
         "Effect": "Allow",
         "Principal": {
           "Service": "ec2.amazonaws.com"
         },
         "Action": "sts:AssumeRole"
       }
     ]
}</code></pre> 
</div> 
<h3>Step 2: Create a new Windows instance from the EC2 console</h3> 
<p>With this role in place, you can now join a Windows instance to your domain via the EC2 launch wizard. For a detailed explanation about how to do this, see <a href="http://docs.aws.amazon.com/AWSEC2/latest/WindowsGuide/ec2-join-aws-domain.html#join-domain-console" rel="noopener" target="_blank">Joining a Domain Using the Amazon EC2 Launch Wizard</a>.</p> 
<p>If you’re instantiating a new instance from the API, however, you will need to create an SSM configuration document and upload it to the SSM service beforehand. We’ll step through that process next.</p> 
<blockquote>
 <p><strong>Note:</strong> The instance will require <a href="https://docs.aws.amazon.com/systems-manager/latest/userguide/setup-create-vpc.html" rel="noopener" target="_blank">internet access</a> to communicate with the SSM service.</p>
</blockquote> 
<div class="wp-caption alignnone" id="attachment_30516" style="width: 610px;">
 <img alt="Figure 7: Configure instance details" class="size-full wp-image-30516" height="306" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2023/08/15/img7-1.png" style="border: 1px solid #bebebe;" width="600" />
 <p class="wp-caption-text" id="caption-attachment-30516">Figure 7: Configure instance details</p>
</div> 
<p>When you create a new Windows instance from the EC2 launch wizard as shown in Figure 7, the wizard automatically creates the SSM configuration document from the information stored in AD Connector. Presently, the EC2 launch wizard doesn’t allow you to specify which organizational unit (OU) you want to deploy the member server into.</p> 
<h3>Step 3: Create an SSM document (for seamlessly joining a server to the domain through the AWS API)</h3> 
<p>If you want to provision new Windows instances from the AWS CLI or API or you want to specify the target OU for your instances, you will need to create an SSM configuration document. The configuration document is a JSON file that contains various parameters used to configure your instances. The following code example is a configuration document for joining a domain.</p> 
<div class="hide-language"> 
 <pre><code class="lang-text">{
	"schemaVersion": "1.0",
	"description": "Sample configuration to join an instance to a domain",
	"runtimeConfig": {
	   "aws:domainJoin": {
	       "properties": {
	          "directoryId": "<span style="color: #ff0000;">d-1234567890</span>",
	          "directoryName": "<span style="color: #ff0000;">test.example.com</span>",
	          "directoryOU": "<span style="color: #ff0000;">OU=test,DC=example,DC=com</span>",
	          "dnsIpAddresses": [
	             "<span style="color: #ff0000;">198.51.100.1</span>",
	             "<span style="color: #ff0000;">198.51.100.2</span>"
	          ]
	       }
	   }
	}
}</code></pre> 
</div> 
<p>In this configuration document:</p> 
<ul> 
 <li><span style="font-family: courier;">directoryId</span> is the ID for the AD Connector you created earlier.</li> 
 <li><span style="font-family: courier;">directoryName</span> is the name of the domain (for example, examplecompany.com).</li> 
 <li><span style="font-family: courier;">directoryOU</span> is the OU for the domain.</li> 
 <li><span style="font-family: courier;">dnsIpAddresses</span> are the IP addresses for the DNS servers you specified when you created the AD Connector.</li> 
</ul> 
<p>For additional information, see <a href="http://docs.aws.amazon.com/ssm/latest/APIReference/aws-domainJoin.html" rel="noopener" target="_blank">aws:domainJoin</a>. When you’re finished creating the file, save it as a JSON file.</p> 
<blockquote>
 <p><strong>Note</strong>: The name of the file has to be at least 1 character and at most 64 characters in length.</p>
</blockquote> 
<h3>Step 4: Upload the configuration document to SSM</h3> 
<p>This step requires that the user have permission to use SSM to configure an instance. If you don’t have a policy that includes these rights, create a new policy by using the following JSON, and assign it to an IAM user or group.</p> 
<div class="hide-language"> 
 <pre><code class="lang-text">{
   "Version": "2012-10-17",
   "Statement": [
     {
       "Effect": "Allow",
       "Action": "ssm:*",
       "Resource": "*"
     }
   ]
}</code></pre> 
</div> 
<p>After you’ve signed in with a user that associates with the SSM IAM policy you created, run the following command from the AWS CLI.</p> 
<div class="hide-language"> 
 <pre><code class="lang-text">aws ssm create-document ‐‐content file:<span style="color: #ff0000;">//path/to/myconfigfile.json</span> ‐‐name <span style="color: #ff0000;">"My_Custom_Config_File"</span></code></pre> 
</div> 
<blockquote>
 <p><strong>Note</strong>: On Linux/Mac systems, you need to add a “/” at the beginning of the path (for example, file:///Users/username/temp).</p>
</blockquote> 
<p>This command uploads the configuration document you created to the SSM service, allowing you to reference it when creating a new Windows instance from either the AWS CLI or the EC2 launch wizard.</p> 
<h2>Conclusion</h2> 
<p>This blog post has shown you how you can simplify account management by federating with your Active Directory for AWS Management Console access. The post also explored how you can enable <em>hybrid IT</em> by using AD Connector to seamlessly join Windows instances to your Active Directory domain. Armed with this information you can create a trust between your Active Directory and AWS. In addition, you now have a quick and simple way to enable single sign-on without needing to replicate identities or deploy additional infrastructure on premises.</p> 
<p>We’d love to hear more about how you are using Directory Service, and welcome any feedback about how we can improve the experience. You can post comments below, or visit the <a href="https://forums.aws.amazon.com/forum.jspa?forumID=180" rel="noopener" target="_blank">Directory Service forum</a> to post comments and questions.</p> 
<p>If you have feedback about this post, submit comments in the Comments section below. If you have questions about this post, start a new thread on the <a href="https://repost.aws/tags/knowledge-center/TAkGIL-2I9RYOFF_luDq9dMQ/aws-directory-service" rel="noopener" target="_blank">AWS Directory Service knowledge Center re:Post</a> or <a href="https://console.aws.amazon.com/support/home" rel="noopener" target="_blank">contact AWS Support</a>.</p> 
<p><strong>Want more AWS Security news? Follow us on <a href="https://twitter.com/AWSsecurityinfo" rel="noopener noreferrer" target="_blank" title="Twitter">Twitter</a>.</strong></p> 
<footer> 
 <div class="blog-author-box"> 
  <div class="blog-author-image"> 
   <img alt="Jeremy Cowan" src="https://d2908q01vomqb2.cloudfront.net/ca3512f4dfa95a03169c5a670a4c91a19b3077b4/2018/09/27/Jeremy_Cowan.png" width="120" /> 
  </div> 
  <h3 class="lb-h4">Jeremy Cowan</h3> 
  <p>Jeremy is a Specialist Solutions Architect for containers at AWS, although his family thinks he sells “cloud space”. Prior to joining AWS, Jeremy worked for several large software vendors, including VMware, Microsoft, and IBM. When he’s not working, you can usually find on a trail in the wilderness, far away from technology.</p> 
  <p></p>
 </div> 
 <div class="blog-author-box"> 
  <div class="blog-author-image"> 
   <img alt="Bright Dike" class="alignnone size-full wp-image-30508" height="160" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2023/08/15/Bright-Dike.jpg" width="120" /> 
  </div> 
  <h3 class="lb-h4">Bright Dike</h3> 
  <p>Bright is a Solutions Architect with Amazon Web Services. He works with AWS customers and partners to provide guidance assessing and improving their security posture, as well as executing automated remediation techniques. His domains are threat detection, incident response, and security hub.</p> 
  <p></p>
 </div> 
 <div class="blog-author-box"> 
  <div class="blog-author-image"> 
   <img alt="David Selberg" class="alignnone size-full wp-image-30506" height="160" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2023/08/15/David-Selberg.jpg" width="120" /> 
  </div> 
  <h3 class="lb-h4">David Selberg</h3> 
  <p>David is an Enterprise Solutions Architect at AWS who is passionate about helping customers build Well-Architected solutions on the AWS cloud. With a background in cybersecurity, David loves to dive deep on security topics when he’s not creating technical content like the “All Things AWS” Twitch series.</p> 
  <p></p>
 </div> 
 <div class="blog-author-box"> 
  <div class="blog-author-image"> 
   <img alt="Abhra Sinha" class="aligncenter size-full wp-image-30334" height="160" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2023/07/26/Abhra-Sinha.jpg" width="120" /> 
  </div> 
  <h3 class="lb-h4">Abhra Sinha</h3> 
  <p>Abhra is a Toronto-based Enterprise Solutions Architect at AWS. Abhra enjoys being a trusted advisor to customers, working closely with them to solve their technical challenges and help build a secure scalable architecture on AWS. In his spare time, he enjoys photography and exploring new restaurants.</p> 
 </div> 
</footer>
