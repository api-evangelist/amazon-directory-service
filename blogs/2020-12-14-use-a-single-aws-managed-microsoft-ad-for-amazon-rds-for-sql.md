---
title: "Use a single AWS Managed Microsoft AD for Amazon RDS for SQL Server instances in multiple Regions"
url: "https://aws.amazon.com/blogs/security/use-a-single-aws-managed-microsoft-ad-for-amazon-rds-for-sql-server-instances-in-multiple-regions/"
date: "Mon, 14 Dec 2020 20:42:23 +0000"
author: "Jeremy Girven"
feed_url: "https://aws.amazon.com/blogs/security/tag/aws-directory-service/feed/"
---
<p>Many <a href="http://aws.amazon.com/" rel="noopener noreferrer" target="_blank">Amazon Web Services (AWS)</a> customers use Active Directory to centralize user authentication and authorization for a variety of applications and services. For these customers, Active Directory is a critical piece of their IT infrastructure.</p> 
<p>AWS offers <a href="https://aws.amazon.com/directoryservice/" rel="noopener noreferrer" target="_blank">AWS Directory Service for Microsoft Active Directory</a>, also known as AWS Managed Microsoft AD, to provide a highly accessible and resilient Active Directory service that is built on <a href="https://aws.amazon.com/directoryservice/active-directory/" rel="noopener noreferrer" target="_blank">Microsoft Active Directory</a>.</p> 
<p>AWS also offers <a href="https://aws.amazon.com/rds/sqlserver/" rel="noopener noreferrer" target="_blank">Amazon Relational Database Service (Amazon RDS) for SQL Server</a>. Amazon RDS enables you to prioritize application development by managing time-consuming database administration tasks including provisioning, backups, software patching, monitoring, and hardware scaling. If you require Windows authentication with Amazon RDS for SQL Server, Amazon RDS for SQL Server instances need to be integrated with AWS Managed Microsoft AD.</p> 
<p>With the release of AWS Managed Microsoft AD cross-Region support, you only need one distinct AWS Managed Microsoft AD that spans multiple AWS Regions; this simplifies directory management and configuration. Additionally, it simplifies trusts between the AWS Managed Microsoft AD domain and your on-premises domain. Now, only a single trust between your on-premises domain and AWS Managed Microsoft AD domain is required, as compared to the previous pattern of only one AWS Managed Microsoft AD per Region—each of which would require a trust if you wanted to allow on-premises objects access to your AWS Managed Microsoft AD domain. Further, AWS Managed Microsoft AD cross-Region support provides an additional benefit when using your on-premises users and groups with Amazon RDS for SQL Server: You only need a single, one-way, outgoing trust between your multi-Region AWS Managed Microsoft AD and your on-premises domain.</p> 
<p>As detailed in this post, to enable AWS Managed Microsoft AD cross-Region support, you create a new AWS Managed Microsoft AD and extend it to multiple Regions (as shown in Figure 1 below). Once you’ve extended your directory, you deploy an Amazon RDS SQL Server instance in each Region, integrating it to the same directory. Finally, you install SQL Server Management Studio (SSMS) on an instance joined to the AWS Managed Microsoft AD directory. You use that instance to connect to the RDS SQL Server instances using the same domain user account.</p> 
<div class="wp-caption aligncenter" id="attachment_17819" style="width: 833px;">
 <img alt="Figure 1: High level diagram of resources deployed in this post" class="size-full wp-image-17819" height="363" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2020/12/11/Amazon-RDS-SQL-Server-Figure-1.png" width="823" />
 <p class="wp-caption-text" id="caption-attachment-17819">Figure 1: High level diagram of resources deployed in this post</p>
</div> 
<p>The architecture in Figure 1 includes a network connection between the Regions. That connection isn’t required for the AWS Managed Microsoft AD to function. If you don’t require network connectivity between your regions, you can disregard the network link in the diagram. Since you will be using a single <a href="http://aws.amazon.com/ec2" rel="noopener noreferrer" target="_blank">Amazon Elastic Compute Cloud (Amazon EC2)</a> instance in one Region, the network connection is needed between Amazon VPCs in the two Regions to allow that instance to connect to a domain controller in each Region.</p> 
<h2>Prerequisites for AWS Managed Microsoft AD cross-Region Support</h2> 
<ol> 
 <li>An AWS Managed Microsoft AD deployed in a Region of your choice. If you don’t have one already deployed, you can follow the instructions in <a href="https://docs.aws.amazon.com/directoryservice/latest/admin-guide/ms_ad_getting_started_create_directory.html" rel="noopener noreferrer" target="_blank">Create Your AWS Managed Microsoft AD directory</a> to create one. For this post, I recommend that you use us-east-1.</li> 
 <li>The VPC must be peered in order to complete the steps in this blog. <a href="https://docs.aws.amazon.com/vpc/latest/peering/create-vpc-peering-connection.html" rel="noopener noreferrer" target="_blank">Creating and accepting a VPC peering connection</a> has information on how to create a peering connection between Regions. Be aware of <a href="https://docs.aws.amazon.com/vpc/latest/peering/invalid-peering-configurations.html" rel="noopener noreferrer" target="_blank">unsupported VPC peering configurations</a>.</li> 
 <li>A Windows Server instance joined to your managed Active Directory domain. <a href="https://docs.aws.amazon.com/directoryservice/latest/admin-guide/ms_ad_join_instance.html" rel="noopener noreferrer" target="_blank">Join an EC2 Instance to Your AWS Managed Microsoft AD Directory</a> has instructions if you need assistance.</li> 
 <li>Install the Active Directory administration tools onto your domain-joined instance. <a href="https://docs.aws.amazon.com/directoryservice/latest/admin-guide/ms_ad_install_ad_tools.html" rel="noopener noreferrer" target="_blank">Installing the Active Directory Administration Tools</a> has detailed instructions.</li> 
</ol> 
<h2>Extend your AWS Managed Microsoft AD to another Region</h2> 
<p>We’ve made the process to extend your directory to another Region straightforward. There is no cost to add another Region; you only pay for the resources for your directory running in the new Region. See <a href="https://aws.amazon.com/directoryservice/pricing/" rel="noopener noreferrer" target="_blank">here</a> for additional information on pricing changes with new Regions. For example, in this post you will be extending your directory into the us-east-2 region. There will be an additional cost for two new domain controllers. Figure 3 shows the additional cost to extend the directory.</p> 
<p>Let’s walk through the steps of setting up Windows Authentication with Amazon RDS for SQL Server instances in multiple Regions using a single cross-Region AWS Managed Microsoft AD.</p> 
<h3>To extend your directory to another Region:</h3> 
<ol> 
 <li>In the AWS Directory Service console navigation pane, choose <strong>Directories</strong>.<br /> 
  <blockquote>
   <p><strong>Note</strong>: You should see a list of your AWS Managed Microsoft AD directories.</p>
  </blockquote> </li> 
 <li>Choose the <strong>Directory ID</strong> of the directory you want to expand to another Region.</li> 
 <li>Go to the <strong>Directory details</strong> page. In the <strong>Multi-region replication</strong> section, select <strong>Add Region</strong>. <p></p>
  <div class="wp-caption aligncenter" id="attachment_17820" style="width: 1954px;">
   <img alt="Figure 2: Directory details and new multi-Region replication pane" class="size-full wp-image-17820" height="652" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2020/12/11/Amazon-RDS-SQL-Server-Figure-2.png" width="1944" />
   <p class="wp-caption-text" id="caption-attachment-17820">Figure 2: Directory details and new multi-Region replication pane</p>
  </div></li> 
 <li>On the <strong>Add region</strong> page: 
  <ol type="a"> 
   <li>For <strong>Region to add</strong>, select the Region you want to extend your directory to.</li> 
   <li>For <strong>VPC</strong>, select the <a href="http://aws.amazon.com/vpc" rel="noopener noreferrer" target="_blank">Amazon Virtual Private Cloud (Amazon VPC)</a> for the new domain controllers to use.</li> 
   <li>For <strong>Subnets</strong>, select two unique subnets in the Amazon VPC that you selected in the preceding step.</li> 
   <li>Once you have everything to your liking, choose <strong>Add</strong>. 
    <div class="wp-caption aligncenter" id="attachment_17821" style="width: 954px;">
     <img alt="Figure 3: Add a Region" class="size-full wp-image-17821" height="965" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2020/12/11/Amazon-RDS-SQL-Server-Figure-3.png" width="944" />
     <p class="wp-caption-text" id="caption-attachment-17821">Figure 3: Add a Region</p>
    </div> <p>In the background, AWS is provisioning two new AWS managed domain controllers in the Region you selected. It could take up to 2 hours for your directory to become available in the Region.</p></li> 
  </ol> </li> 
</ol> 
<blockquote>
 <p><strong>Note</strong>: Your managed domain controllers in the home Region are fully functional during this process.</p>
</blockquote> 
<ul> 
 <li>On the <strong>Directory details</strong> page, in <strong>Multi-Region replication</strong>, the status should be <strong>Active</strong> when the process has completed. Now you’re ready to deploy your Amazon RDS SQL Server instances.</li> 
</ul> 
<h2>Enable Amazon RDS for SQL Server</h2> 
<p>Integrating Amazon RDS into AWS Managed Microsoft AD is exactly the same process as it was before the cross-Region feature was released. This post goes through that original process with only one change, which is that you select the same directory ID for both Regions.</p> 
<h3>Create an Amazon RDS SQL Server instance in each Region using the same directory</h3> 
<p>The steps for creating an Amazon RDS SQL Server instance in each Region are the same. The following process will create the first instance. Once you’ve completed the process, you change the <a href="http://aws.amazon.com/console" rel="noopener noreferrer" target="_blank">AWS Management Console</a> Region to the Region you extended your directory to and repeat the process.</p> 
<h4>To create an Amazon RDS SQL Server instance:</h4> 
<ol> 
 <li>In the AWS Managed Microsoft AD directory primary Region, go to the Amazon RDS console navigation pane and choose <strong>Create database</strong>.</li> 
 <li>Choose <strong>Microsoft SQL Server</strong>.</li> 
 <li>You can leave the default values, except for the following settings: 
  <ol type="a"> 
   <li>Under <strong>Settings</strong> select <strong>Master</strong> and <strong>Confirm password</strong>.</li> 
   <li>Under <strong>Connectivity</strong>, expand <strong>Additional connectivity configuration</strong>: 
    <ol type="i"> 
     <li>Choose <strong>Create new</strong> to create a new VPC security group.</li> 
     <li>Enter a name in <strong>New VPC security group name</strong>.</li> 
     <li>Select <strong>No preference</strong> for <strong>Availability Zone</strong>.</li> 
     <li>Enter <span style="font-family: courier;">1433</span> for Database port.</li> 
    </ol> <p></p>
    <div class="wp-caption aligncenter" id="attachment_17822" style="width: 770px;">
     <img alt="Figure 4: Connectivity settings" class="size-full wp-image-17822" height="924" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2020/12/11/Amazon-RDS-SQL-Server-Figure-4.png" width="760" />
     <p class="wp-caption-text" id="caption-attachment-17822">Figure 4: Connectivity settings</p>
    </div></li> 
  </ol> </li> 
 <li>Select the <strong>Enable Microsoft SQL Server Windows authentication</strong> check box and then choose <strong>Browse Directory</strong>. <p></p>
  <div class="wp-caption aligncenter" id="attachment_17823" style="width: 769px;">
   <img alt="Figure 5: Enable Microsoft SQL Server Windows authentication selected" class="size-full wp-image-17823" height="263" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2020/12/11/Amazon-RDS-SQL-Server-Figure-5.png" width="759" />
   <p class="wp-caption-text" id="caption-attachment-17823">Figure 5: Enable Microsoft SQL Server Windows authentication selected</p>
  </div></li> 
 <li>Select your directory and select <strong>Choose</strong>. <p></p>
  <div class="wp-caption aligncenter" id="attachment_17824" style="width: 827px;">
   <img alt="Figure 6: Select a directory" class="size-full wp-image-17824" height="336" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2020/12/11/Amazon-RDS-SQL-Server-Figure-6.png" width="817" />
   <p class="wp-caption-text" id="caption-attachment-17824">Figure 6: Select a directory</p>
  </div></li> 
 <li>Choose <strong>Create database</strong>.</li> 
 <li>Repeat these steps in your expanded Region. Note that the Directory ID will be the same for both Regions. You can complete the next section while your Amazon RDS SQL instances are provisioning.</li> 
</ol> 
<h3>Create an Active Directory user and group to delegate SQL administrative rights</h3> 
<p>The following steps walk you through the process of creating an Active Directory user and group for delegation. Following this process, you add the user to the group you just created and to the AWS Delegated Server Administrators group.</p> 
<h4>To create a user and group:</h4> 
<ol> 
 <li>Log in to the domain-joined instance with a domain user account that has permissions to create Active Directory users and groups.</li> 
 <li>Choose <strong>Start</strong>, enter <span style="font-family: courier;">dsa.msc</span>, and press <strong>Enter</strong>.</li> 
 <li>In <strong>Active Directory Users and Computers</strong>, right-click on the <strong>Users</strong> OU, select <strong>New</strong>, and then <strong>Group</strong>. The <strong>New Object – Group</strong> window pops up. 
  <ol type="a"> 
   <li>Fill in the <strong>Group name</strong> boxes with your choice of name.</li> 
   <li>For <strong><a href="https://docs.microsoft.com/en-us/windows/security/identity-protection/access-control/active-directory-security-groups#group-scope" rel="noopener noreferrer" target="_blank">Group Scope</a></strong>, select <strong>Domain local</strong>.</li> 
   <li>For <strong>Group type</strong>, select <strong>Security</strong>.</li> 
   <li>Choose <strong>OK</strong>.</li> 
  </ol> </li> 
 <li>In <strong>Active Directory Users and Computers</strong>, right-click on your Users OU and select <strong>New</strong> and then <strong>User</strong>. The <strong>New Object – User</strong> window pops up. 
  <ol type="a"> 
   <li>Fill in the boxes with your choice of information, and then choose <strong>Next</strong>.</li> 
   <li>Enter your choice of password and clear <strong>User must change password at next logon</strong>, then choose <strong>Next</strong>.</li> 
   <li>On the confirmation page, choose <strong>Finish</strong>.</li> 
  </ol> </li> 
 <li>Double-click on the user you just created. The user account properties window appears. 
  <ol type="a"> 
   <li>Select the <strong>Member of</strong> tab.</li> 
   <li>Choose <strong>Add</strong>.</li> 
   <li>Enter the name of the group that you previously created and choose <strong>Check Names</strong>. Next, enter <span style="font-family: courier;">AWS Delegated Server Administrators</span> and choose <strong>Check Names</strong> again. If you do not receive any error, choose <strong>OK</strong>, and then <strong>OK</strong> again.</li> 
  </ol> </li> 
 <li>The Member of tab for the user should include the two groups you just added. Choose <strong>OK</strong> to close the properties page.</li> 
</ol> 
<h2>Delegate SQL Server permissions in each Region using the Active Directory group you just created</h2> 
<p>The following steps guide you through the process of modifying the Amazon RDS SQL security group, installing SQL Server Management Studio (SSMS), and delegating permission in SQL to your Active Directory group.</p> 
<h3>Modify the Amazon RDS SQL security group</h3> 
<p>In these next steps, you modify the security group you created with your Amazon RDS instances, allowing your Windows Server instance to connect to the Amazon RDS SQL Server instances over port 1433.</p> 
<h4>To modify the security group:</h4> 
<ol> 
 <li>From the <a href="https://www.google.com/url?sa=t&amp;rct=j&amp;q=&amp;esrc=s&amp;source=web&amp;cd=&amp;cad=rja&amp;uact=8&amp;ved=2ahUKEwi_3PP0q9bsAhWSYDUKHSohDK8QFjAAegQIBBAC&amp;url=https%3A%2F%2Fconsole.aws.amazon.com%2Fec2%2Fv2%2Fhome&amp;usg=AOvVaw1GOOlA2ZJcJD_cVZwOauat" rel="noopener noreferrer" target="_blank">Amazon Elastic Compute Cloud (Amazon EC2)</a> console, select <strong>Security Groups</strong> under the <strong>Network &amp; Security</strong> navigation section.</li> 
 <li>Select the new Amazon RDS SQL security group that was created with your Amazon RDS SQL instance and select <strong>Edit inbound rules</strong>.</li> 
 <li>Choose <strong>Add rule</strong> and enter the following: 
  <ol type="a"> 
   <li><strong>Type</strong> – Select <strong>Custom TCP</strong>.</li> 
   <li><strong>Protocol</strong> – Select <strong>TCP</strong>.</li> 
   <li><strong>Port range</strong> – Enter <span style="font-family: courier;">1433</span>.</li> 
   <li>Source – Select <strong>Custom</strong>.</li> 
   <li>Enter the private IP of your instance with a <span style="font-family: courier;">/32</span>. An example would be <span style="font-family: courier;">10.0.0.10/32</span>.</li> 
  </ol> </li> 
 <li>Choose <strong>Save rules</strong>. <p></p>
  <div class="wp-caption aligncenter" id="attachment_17827" style="width: 1932px;">
   <img alt="Figure 7: Create a security group rule" class="size-full wp-image-17827" height="496" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2020/12/11/Amazon-RDS-SQL-Server-Figure-7.png" width="1922" />
   <p class="wp-caption-text" id="caption-attachment-17827">Figure 7: Create a security group rule</p>
  </div></li> 
 <li>Repeat these steps on the security group of your other Amazon RDS SQL instance in the other Region.</li> 
</ol> 
<h3>Install SQL Server Management Studio</h3> 
<p>All of the steps after the first are done on the Windows Server instance from Prerequisite 3.</p> 
<h4>To install SMMS:</h4> 
<ol> 
 <li>On your local computer, download <a href="https://docs.microsoft.com/en-us/sql/ssms/download-sql-server-management-studio-ssms?view=sql-server-ver15" rel="noopener noreferrer" target="_blank">SQL Server Management Studio (SSMS)</a>.<br /> 
  <blockquote>
   <p><strong>Note</strong>: If desired, you can disable IE Enhanced Security Configuration and download directly to the Windows Server instance using IE or any other browser, and skip to step 3.</p>
  </blockquote> </li> 
 <li>RDP into your Windows Server instance and copy <span style="font-family: courier;">SSMS-Setup-ENU.exe</span> to your RDP session.</li> 
 <li>Run the file on your Windows Server instance.</li> 
 <li>Choose <strong>Install</strong>. <p></p>
  <div class="wp-caption aligncenter" id="attachment_17829" style="width: 700px;">
   <img alt="Figure 8: Install SMMS" class="size-full wp-image-17829" height="595" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2020/12/11/Amazon-RDS-SQL-Server-Figure-8.png" width="690" />
   <p class="wp-caption-text" id="caption-attachment-17829">Figure 8: Install SMMS</p>
  </div></li> 
 <li>It might take a few minutes to install. When complete, choose <strong>Close</strong>.</li> 
</ol> 
<h3>Delegate permissions in SSMS</h3> 
<p>All of the following steps are performed on the Windows Server instance from Prerequisite 3. Log in to the Amazon RDS SQL instance using the SQL master user account. Next, create a SQL login for the Active Directory group you created previously and give it elevated permission to the Amazon RDS SQL instance.</p> 
<h4>To delegate permissions:</h4> 
<ol> 
 <li>Start SMMS.</li> 
 <li>On the <strong>Connect to Server</strong> window, enter or select: 
  <ol type="a"> 
   <li><strong>Server name</strong> – Your Amazon RDS SQL Server endpoint.</li> 
   <li><strong>Authentication</strong> – Select <strong>SQL Server Authentication</strong>.</li> 
   <li><strong>Login</strong> – Enter the master user name you used when you launched your Amazon RDS SQL instance. The default is <span style="font-family: courier;">admin</span>.</li> 
   <li><strong>Password</strong> – Enter the password for the master user name.</li> 
   <li>Choose <strong>Connect</strong>.</li> 
  </ol> <p></p>
  <div class="wp-caption aligncenter" id="attachment_17831" style="width: 483px;">
   <img alt="Figure 9: Connect to server" class="size-full wp-image-17831" height="309" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2020/12/11/Amazon-RDS-SQL-Server-Figure-9.png" width="473" />
   <p class="wp-caption-text" id="caption-attachment-17831">Figure 9: Connect to server</p>
  </div></li> 
 <li>In SMMS, Choose <strong>New Query</strong> at the top of the window. <p></p>
  <div class="wp-caption aligncenter" id="attachment_17832" style="width: 329px;">
   <img alt="Figure 10: New query" class="size-full wp-image-17832" height="74" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2020/12/11/Amazon-RDS-SQL-Server-Figure-10.png" width="319" />
   <p class="wp-caption-text" id="caption-attachment-17832">Figure 10: New query</p>
  </div></li> 
 <li>In the query window, enter the following query. Replace <em><span style="font-family: courier;"><span style="color: #ff0000;">&lt;CORP\SQL-Admins&gt;</span></span></em> with the name of the group you created earlier. 
  <div class="hide-language"> 
   <pre><code class="lang-text">CREATE LOGIN [<em><span style="font-family: courier;"><span style="color: #ff0000;">&lt;CORP\SQL-Admins&gt;</span></span></em>] FROM WINDOWS WITH DEFAULT_DATABASE = [master],
   DEFAULT_LANGUAGE = [us_english];
</code></pre> 
  </div> <p></p>
  <div class="wp-caption aligncenter" id="attachment_17834" style="width: 591px;">
   <img alt="Figure 11: Query SQL database" class="size-full wp-image-17834" height="55" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2020/12/11/Amazon-RDS-SQL-Server-Figure-11.png" width="581" />
   <p class="wp-caption-text" id="caption-attachment-17834">Figure 11: Query SQL database</p>
  </div></li> 
 <li>Choose <strong>Execute</strong> on the menu bar. You should see a <span style="font-family: courier;">Commands completed successfully</span> message. <p></p>
  <div class="wp-caption aligncenter" id="attachment_17836" style="width: 584px;">
   <img alt="Figure 12: Commands completed successfully" class="size-full wp-image-17836" height="144" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2020/12/11/Amazon-RDS-SQL-Server-Figure-12.png" width="574" />
   <p class="wp-caption-text" id="caption-attachment-17836">Figure 12: Commands completed successfully</p>
  </div></li> 
 <li>Next, navigate to the <strong>Logins</strong> directory on the navigation page. Right-click on the group you added with the SQL command in step 5 and select <strong>Properties</strong>. <p></p>
  <div class="wp-caption aligncenter" id="attachment_17856" style="width: 386px;">
   <img alt="Figure 13: Open group properties" class="size-full wp-image-17856" height="250" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2020/12/11/Amazon-RDS-SQL-Server-Figure-13r.png" width="376" />
   <p class="wp-caption-text" id="caption-attachment-17856">Figure 13: Open group properties</p>
  </div></li> 
 <li>Select <strong>Server Roles</strong> and select the <strong>processadmin</strong> and <strong>setupadmin</strong> checkboxes. Then choose <strong>OK</strong>. <p></p>
  <div class="wp-caption aligncenter" id="attachment_17838" style="width: 704px;">
   <img alt="Figure 14: Configure server roles" class="size-full wp-image-17838" height="656" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2020/12/11/Amazon-RDS-SQL-Server-Figure-14.png" width="694" />
   <p class="wp-caption-text" id="caption-attachment-17838">Figure 14: Configure server roles</p>
  </div></li> 
 <li>You can log off from the instance. For the next steps, you log in to the instance using the user account you created previously.</li> 
 <li>Repeat these steps on the Amazon RDS SQL instance in the other Region.</li> 
</ol> 
<h3>Connect to the Amazon RDS SQL Server with the same Active Directory user in both Regions</h3> 
<p>All of the steps are performed on the Windows Server instance from Prerequisite 3. You must log in to the instance using the account you created earlier. You then log in to the Amazon RDS SQL instance using Windows authentication with that account.</p> 
<ol> 
 <li>Log in to the instance with the user account you created earlier.</li> 
 <li>Start <strong>SSMS</strong>.</li> 
 <li>On the <strong>Connect to Server</strong> window, enter or select: 
  <ol type="a"> 
   <li><strong>Server name</strong>: Your Amazon RDS SQL Server endpoint.</li> 
   <li><strong>Authentication</strong>: Select <strong>Windows Authentication</strong>.</li> 
   <li>Choose <strong>Connect</strong>.</li> 
  </ol> <p></p>
  <div class="wp-caption aligncenter" id="attachment_17839" style="width: 483px;">
   <img alt="Figure 15: Connect to server" class="size-full wp-image-17839" height="308" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2020/12/11/Amazon-RDS-SQL-Server-Figure-15.png" width="473" />
   <p class="wp-caption-text" id="caption-attachment-17839">Figure 15: Connect to server</p>
  </div></li> 
 <li>You should be logged in to SSMS. If you aren’t logged in, make sure you added your user account to the group you created earlier and try again.</li> 
 <li>Repeat these steps using the other Amazon RDS SQL instance endpoint for the server name. You should be able to connect to both Amazon RDS SQL instances using the same user account.</li> 
</ol> 
<h2>Summary</h2> 
<p>In this post, you extended your <a href="https://aws.amazon.com/directoryservice/active-directory/" rel="noopener noreferrer" target="_blank">AWS Managed Microsoft AD</a> into a new Region. You then deployed <a href="https://aws.amazon.com/rds/sqlserver/" rel="noopener noreferrer" target="_blank">Amazon RDS for SQL Server</a> in multiple Regions attached to the same AWS Managed Microsoft AD directory. You then tested authentication to both Amazon RDS SQL instances using the same Active Directory user.</p> 
<p>To learn more about using AWS Managed Microsoft AD or AD Connector, visit the <a href="https://docs.aws.amazon.com/directory-service/index.html" rel="noopener noreferrer" target="_blank">AWS Directory Service documentation</a>. For general information and pricing, see the <a href="https://aws.amazon.com/directoryservice/" rel="noopener noreferrer" target="_blank">AWS Directory Service home page</a>. If you have comments about this blog post, submit a comment in the <strong>Comments</strong> section below. If you have implementation or troubleshooting questions, start a new thread on the <a href="https://forums.aws.amazon.com/forum.jspa?forumID=180" rel="noopener noreferrer" target="_blank">AWS Directory Service forum</a> or contact <a href="https://console.aws.amazon.com/support/home?" rel="noopener noreferrer" target="_blank">AWS Support</a>.</p> 
<footer> 
 <div class="blog-author-box"> 
  <div class="blog-author-image">
   <img alt="Author" class="aligncenter size-full wp-image-17843" height="160" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2020/12/11/Jeremy-Girven-Author.jpg" width="120" />
  </div> 
  <h3 class="lb-h4">Jeremy Girven</h3> 
  <p>Jeremy is a Solutions Architect specializing in Microsoft workloads on AWS. He has over 15 years of experience with Microsoft Active Directory and over 23 years of industry experience. One of his fun projects is using SSM to automate the Active Directory build processes in AWS. To see more please check out the <a href="https://aws.amazon.com/quickstart/architecture/active-directory-ds/" rel="noopener noreferrer" target="_blank">Active Directory AWS QuickStart</a>.</p> 
 </div> 
</footer>
