---
title: "Everything you wanted to know about trusts with AWS Managed Microsoft AD"
url: "https://aws.amazon.com/blogs/security/everything-you-wanted-to-know-about-trusts-with-aws-managed-microsoft-ad/"
date: "Tue, 16 Nov 2021 20:02:07 +0000"
author: "Jeremy Girven"
feed_url: "https://aws.amazon.com/blogs/security/tag/aws-directory-service/feed/"
---
<p>Many <a href="http://aws.amazon.com/" rel="noopener noreferrer" target="_blank">Amazon Web Services (AWS)</a> customers use Active Directory to centralize user authentication and authorization for a variety of applications and services. For these customers, Active Directory is a critical piece of their IT infrastructure. AWS offers <a href="https://aws.amazon.com/directoryservice/" rel="noopener noreferrer" target="_blank">AWS Directory Service for Microsoft Active Directory</a>, also known as AWS Managed Microsoft AD, to provide a highly available and resilient Active Directory service.</p> 
<p>One of the most common AWS Managed Microsoft AD use cases is for customers who need to integrate their on-premises Active Directory domain or forest with AWS services like <a href="https://aws.amazon.com/rds/" rel="noopener noreferrer" target="_blank">Amazon Relational Database Service (Amazon RDS)</a>, <a href="https://aws.amazon.com/fsx/" rel="noopener noreferrer" target="_blank">Amazon FSx</a>, <a href="https://aws.amazon.com/workspaces" rel="noopener noreferrer" target="_blank">Amazon WorkSpaces</a>, and other <a href="https://docs.aws.amazon.com/directoryservice/latest/admin-guide/ms_ad_app_compatibility.html" rel="noopener noreferrer" target="_blank">AWS applications and services</a>. This type of integration can require a trust relationship. When it comes to trusts, there are some common misconceptions about what happens and doesn’t happen when a trust is created.</p> 
<p>In this post, I’m going to dive deep into various aspects of Active Directory trusts and debunk some common myths along the way. This post will cover the following areas:</p> 
<ul> 
 <li><a href="#_Starting_with_Kerberos">Kerberos authentication across trusts</a></li> 
 <li><a href="#_Trust_Transitivity,_Direction,">Trust fundamentals</a></li> 
 <li><a href="#_Trust_Creation_Process">Trust creation process overview</a></li> 
 <li><a href="#_Common_trust_scenarios">Common trust scenarios</a></li> 
 <li><a href="#_Common_Trusts_Myths">Trust myths and misconceptions</a></li> 
 <li><a href="#_Troubleshooting_Trusts">Troubleshooting trusts</a></li> 
</ul> 
<p id="_Starting_with_Kerberos"></p> 
<h2>Starting with Kerberos</h2> 
<p>The first part of understanding how trusts work is to understand how authentication flows across a trust, particularly with Kerberos. Kerberos is a subject that, on the surface, is simple enough, but can quickly become much more complex. This post isn’t going to go into detail about Kerberos in Microsoft Windows. If you wish to look further into the topic, see <a href="https://docs.microsoft.com/en-us/previous-versions/windows/it-pro/windows-server-2003/cc772815(v=ws.10)?redirectedfrom=MSDN" rel="noopener noreferrer" target="_blank">the Microsoft Kerberos documentation</a>. In this post, I’m just going to give you an overview of how Kerberos authentication works across trusts.<br /> &nbsp;<br /> </p>
<div class="wp-caption aligncenter" id="attachment_22913" style="width: 660px;">
 <img alt="Figure 1: Kerberos authentication across trusts" class="size-full wp-image-22913" height="671" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2021/11/12/Trusts-AWS-Managed-Microsoft-AD-1.png" width="650" />
 <p class="wp-caption-text" id="caption-attachment-22913">Figure 1: Kerberos authentication across trusts</p>
</div>
<p></p> 
<p>If you only remember one thing about Kerberos and trust, it should be referrals. Let’s look at the workflow in Figure 1, which shows a user from Domain A who is logged into a computer in Domain A and wants to access an Amazon FSx file share in Domain B. For simplicity’s sake, I’ll say there is a two-way trust between Domains A and B.</p> 
<blockquote>
 <p><strong>Note</strong>: When a trust is integrated with AWS Managed Microsoft AD, you need to enable <a href="https://docs.microsoft.com/en-us/previous-versions/windows/it-pro/windows-2000-server/cc961961(v=technet.10)?redirectedfrom=MSDN" rel="noopener noreferrer" target="_blank">Kerberos preauthentication</a> for accounts that traverse the trusts. Disabling Kerberos preauthentication isn’t recommended, because a malicious user can directly send dummy requests for authentication. The key distribution center (KDC) will return an encrypted Ticket-Granting Ticket (TGT), which the malicious user can brute force offline. See <a href="https://social.technet.microsoft.com/wiki/contents/articles/23559.kerberos-pre-authentication-why-it-should-not-be-disabled.aspx" rel="noopener noreferrer" target="_blank">Kerberos Pre-Authentication: Why It Should Not Be Disabled</a> for more details.</p>
</blockquote> 
<p>The steps of the Kerberos authentication process over trusts are as follows:</p> 
<p><strong>1. Kerberos authentication service request (KRB_AS_REQ): </strong>The client contacts the authentication service (AS) of the KDC (which is running on a domain controller) for Domain A, which the client is a member of, for a short-lived ticket called a <em>Ticket-Granting Ticket (TGT)</em>. The default lifetime of the TGT is 10 hours. For Windows clients this happens at logon, but Linux clients might need to run a <span style="font-family: courier;">kinit</span> command.</p> 
<p><strong>2. Kerberos authentication service response (KRB_AS_REP): </strong>The AS constructs the TGT and creates a session key that the client can use to encrypt communication with the ticket-granting service (TGS). At the time that the client receives the TGT, the client has not been granted access to any resources, even to resources on the local computer.</p> 
<p><strong>3. Kerberos ticket-granting service request (KRB_TGS_REQ): </strong>The user’s Kerberos client sends a KRB_TGS_REQ message to a local KDC in Domain A, specifying <span style="font-family: courier;">fsx@domainb</span> as the target. The Kerberos client compares the location with its own workstation’s domain. Because these values are different, the client sets a flag in the KDC <strong>Options</strong> field of the KRB_TGS_REQ message for NAME_CANONICALIZE, which indicates to the KDC that the server might be in another realm (domain).</p> 
<p><strong>4. Kerberos ticket-granting service response (KRB_TGS_REP): </strong>The user’s local KDC (for Domain A) receives the KRB_TGS_REQ and sends back a <em>TGT referral ticket</em> for Domain B. The TGT is issued for the next intervening domain along the shortest path to Domain B. The TGT also has a referral flag set, so that the KDC will be informed that the KRB_TGS_REQ is coming from another realm. This flag also tells the KDC to fill in the <strong>Transited Realms</strong> field. The referral ticket is encrypted with the interdomain key that is decrypted by Domain B’s TGS.</p> 
<blockquote>
 <p><strong>Note:</strong> When a trust is established between domains or forests, an interdomain key based on the trust password becomes available for authenticating KDC functions and is used to encrypt and decrypt Kerberos tickets.</p>
</blockquote> 
<p><strong>5. Kerberos ticket-granting service request (KRB_TGS_REQ): </strong>The user’s Kerberos client sends a KRB_TGS_REQ along with the TGT it received from the Domain A KDC to a KDC in Domain B.</p> 
<p><strong>6. Kerberos ticket-granting service response (KRB_TGS_REP): </strong>The TGS in Domain B examines the TGT and the authenticator. If these are acceptable, the TGS creates a service ticket. The client’s identity is taken from the TGT and copied to the service ticket. Then the ticket is sent to the client.</p> 
<p>For more details on the authenticator, see <a href="https://docs.microsoft.com/en-us/previous-versions/windows/it-pro/windows-server-2003/cc772815(v=ws.10)#the-authenticator" rel="noopener noreferrer" target="_blank">How the Kerberos Version 5 Authentication Protocol Works</a>.</p> 
<p><strong>7. Application server service request (KRB_TGS_REQ):</strong> After the client has the service ticket, the client sends the ticket and a new authenticator to the target server, requesting access. The server will decrypt the ticket, validate the authenticator, and (for Windows services), create an access token for the user based on the SIDs in the ticket.</p> 
<p><strong>8. Application server service response (KRB_TGS_REP): </strong>Optionally, the client might request that the target server verify its own identity. This is called mutual authentication. If mutual authentication is requested, the target server takes the client computer’s timestamp from the authenticator, encrypts it with the session key the TGS provided for client-target server messages, and sends it to the client.</p> 
<p id="_Trust_Transitivity,_Direction,"></p> 
<h2>The basics of trust transitivity, direction, and types</h2> 
<p>Let’s start off by defining a trust. Active Directory trusts are a relationship between domains, which makes it possible for users in one domain to be authenticated by a domain controller in the other domain. Authenticated users, if given proper permissions, can access resources in the other domain.</p> 
<p>Active Directory Domain Services supports four types of trusts: External (Domain), Forest, Realm, and Shortcut. Out of those four types of trusts, <a href="https://aws.amazon.com/directoryservice/" rel="noopener noreferrer" target="_blank">AWS Managed Microsoft AD</a> supports the External (Domain) and Forest trust types. I’ll focus on External (Domain) and Forest trust types for this post.</p> 
<h3>Transitivity: What is it?</h3> 
<p>Before I dive into the types of trusts, it’s important to understand the concept of transitivity in trusts. A trust that is transitive allows authentication to flow through other domains (Child and Trees) in the trusted forests or domains. In contrast, a non-transitive trust is a point-to-point trust that allows authentication to flow exclusively between the trusted domains.<br /> &nbsp;<br /> </p>
<div class="wp-caption aligncenter" id="attachment_22914" style="width: 372px;">
 <img alt="Figure 2: Forest trusts between the Example.local and Example.com forests" class="size-full wp-image-22914" height="239" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2021/11/12/Trusts-AWS-Managed-Microsoft-AD-2.jpg" width="362" />
 <p class="wp-caption-text" id="caption-attachment-22914">Figure 2: Forest trusts between the Example.local and Example.com forests</p>
</div>
<p></p> 
<p>Don’t worry about the trust types at this point, because I’ll cover those shortly. The example in Figure 2 shows a Forest trust between Example.com and Example.local. The Example.local forest has a child domain named Child. With a transitive trust, users from the Example.local and Child.Example.local domain can be authenticated to resources in the Example.com domain.</p> 
<p>If Figure 2 has an External trust, only users from Example.local can be authenticated to resources in the Example.com domain. Users from Child.Example.local cannot traverse the trust to access resources in the Example.com domain.</p> 
<h3>Trust direction</h3> 
<p><em>Two-way</em> trusts are bidirectional trusts that allow authentication referrals from either side of the trust to give users access resources in either domain or forest. If you look in the Active Directory Domains and Trusts area of the <a href="https://docs.microsoft.com/en-us/troubleshoot/windows-server/system-management-components/what-is-microsoft-management-console" rel="noopener noreferrer" target="_blank">Microsoft Management Console (MMC)</a>, which provides consoles to manage the hardware, software, and network components of Microsoft Windows operating system, you can see both an incoming and an outgoing trust for the trusted domain.</p> 
<p><em>One-way</em> trusts are a single-direction trust that allows authentication referrals from one side of the trust only. A one-way trust is either outgoing or incoming, but not both (that would be a two-way trust).</p> 
<ul> 
 <li>An <em>outgoing</em> trust allows users from the trusted domain (Example.com) to authenticate in this domain (Example.local).</li> 
 <li>An <em>incoming</em> trust allows users from this domain (Example.local) to authenticate in the trusted domain (Example.com).</li> 
</ul> 
<p>&nbsp;<br /> </p>
<div class="wp-caption aligncenter" id="attachment_22915" style="width: 338px;">
 <img alt="Figure 3: One-way trust direction" class="size-full wp-image-22915" height="113" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2021/11/12/Trusts-AWS-Managed-Microsoft-AD-3.png" width="328" />
 <p class="wp-caption-text" id="caption-attachment-22915">Figure 3: One-way trust direction</p>
</div>
<p></p> 
<p>Let’s use a diagram to further explain this concept. Figure 3 shows a one-way trust between Example.com and Example.local. This an outgoing trust from Example.com and an incoming trust on Example.local. Users from Example.local can authenticate and, if given proper permissions, access resources in Example.com. Users from Example.com cannot access or authenticate to resources in Example.local.</p> 
<h3>Trust types</h3> 
<p>In this section of the post, I’ll examine the various types of Active Directory trusts and their capabilities.</p> 
<h4>External trusts</h4> 
<p>This trust type is used to share resources between two domains. These can be individual domains within or external to a forest. Think of this as a point-to-point trust between two domains. See <a href="https://docs.microsoft.com/en-us/previous-versions/windows/it-pro/windows-server-2008-r2-and-2008/cc732859(v=ws.10)" rel="noopener noreferrer" target="_blank">Understanding When to Create an External Trust</a> for more details on this trust type.</p> 
<ul> 
 <li>Transitivity: <strong>Non-transitive </strong></li> 
 <li>Direction: <strong>One-way or two-way</strong></li> 
 <li>Authentication types: <strong>NTLM Only*</strong> (Kerberos is possible with caveats; see <a href="https://docs.microsoft.com/en-us/previous-versions/windows/it-pro/windows-server-2003/dd560679(v=ws.10)?redirectedfrom=MSDN#external-trusts-vs-forest-trusts" rel="noopener noreferrer" target="_blank">the Microsoft Windows Server documentation</a> for details)</li> 
 <li>AWS Managed Microsoft AD support: <strong>Yes</strong></li> 
</ul> 
<h4>Forest trusts</h4> 
<p>This trust type is used to share resources between two forests. This is the preferred trust model, because it works fully with Kerberos without any caveats. See <a href="https://docs.microsoft.com/en-us/previous-versions/windows/it-pro/windows-server-2008-r2-and-2008/cc771397(v=ws.10)" rel="noopener noreferrer" target="_blank">Understanding When to Create a Forest Trust</a> for more details.</p> 
<ul> 
 <li>Transitivity: <strong>Transitive </strong></li> 
 <li>Direction: <strong>One-way or two-way</strong></li> 
 <li>Authentication types: <strong>Kerberos and NTLM</strong></li> 
 <li>AWS Managed Microsoft AD support: <strong>Yes</strong></li> 
</ul> 
<h4>Realm trusts</h4> 
<p>This trust type is used to form a trust relationship between a non-Windows Kerberos realm and an Active Directory domain. See <a href="https://docs.microsoft.com/en-us/previous-versions/windows/it-pro/windows-server-2008-r2-and-2008/cc731297(v=ws.10)" rel="noopener noreferrer" target="_blank">Understanding When to Create a Realm Trust</a> for more details.</p> 
<ul> 
 <li>Transitivity: <strong>Non-transitive or transitive </strong></li> 
 <li>Direction: <strong>One-way or two-way</strong></li> 
 <li>Authentication types: <strong>Kerberos Only</strong></li> 
 <li>AWS Managed Microsoft AD support: <strong>No</strong></li> 
</ul> 
<h4>Shortcut trusts</h4> 
<p>This trust type is used to shorten the authentication path between domains within complex forests. See <a href="https://docs.microsoft.com/en-us/previous-versions/windows/it-pro/windows-server-2008-r2-and-2008/cc754538(v=ws.10)" rel="noopener noreferrer" target="_blank">Understanding When to Create a Shortcut Trust</a> for more details.</p> 
<ul> 
 <li>Transitivity: <strong>Transitive </strong></li> 
 <li>Direction: <strong>One-way or two-way</strong></li> 
 <li>Authentication types: <strong>Kerberos and NTLM</strong></li> 
 <li>AWS Managed Microsoft AD support: <strong>No</strong></li> 
</ul> 
<h3>User Principal Name suffixes</h3> 
<p>The default User Principal Name (UPN) suffix for a user account is the Domain Name System (DNS) domain name of the domain where the user account resides. In AWS Managed Microsoft AD and self-managed AD, alternative UPN suffixes are added to simplify administration and user logon processes by providing a single UPN suffix for all users. The UPN suffix is used within the Active Directory forest, and is not required to be a valid DNS domain name. See <a href="https://docs.microsoft.com/en-us/previous-versions/windows/it-pro/windows-server-2008-R2-and-2008/cc772007(v=ws.11)?redirectedfrom=MSDN" rel="noopener noreferrer" target="_blank">Adding User Principal Name Suffixes</a> for the process to add UPN suffixes to a forest.</p> 
<p>For example, if your domain is Example.local but you want your users to sign in with what appears to be another domain name (such as ExampleSuffix.local), you would need to add a new UPN suffix to the domain. Figure 4 shows a user being created with an alternate UPN suffix.<br /> &nbsp;<br /> </p>
<div class="wp-caption aligncenter" id="attachment_22916" style="width: 447px;">
 <img alt="Figure 4: UPN selection on object creation" class="size-full wp-image-22916" height="378" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2021/11/12/Trusts-AWS-Managed-Microsoft-AD-4.png" width="437" />
 <p class="wp-caption-text" id="caption-attachment-22916">Figure 4: UPN selection on object creation</p>
</div>
<p></p> 
<p>If you’re logged into a Windows system, you can use the <span style="font-family: courier;">whoami /upn</span> command to see the UPN of the current user.</p> 
<h3>Forest trusts and name suffix routing</h3> 
<p>Name suffix routing manages how authentication requests are routed across forest trusts. A unique name suffix is a name suffix within a forest, such as a UPN suffix or DNS forest or domain tree name, that isn’t subordinate to any other name suffix. For example, the DNS forest name Example.com is a unique name suffix within the example.com forest.</p> 
<p>All names that are subordinate to unique name suffixes are routed implicitly. For example, if your forest root is named Example.local, authentication requests for all child domains of Example.local (Child.Example.local) will be routed because the child domains are subordinate to the Example.local name suffix. If you want to exclude members of a child domain from authenticating in the specified forest, you can disable name suffix routing for that name. You can also disable routing for the forest name itself, if necessary. With domain trees and additional UPN suffixes, name suffix routing by default is disabled and must be enabled if those suffixes are to be able to traverse the trust.</p> 
<blockquote>
 <p><strong>Note:</strong> In AWS Managed Microsoft AD, customers don’t have the ability to create or modify trusts by using the native Microsoft tools. If you need a name suffix route enabled for your trust, open a support case with Premium Support.</p>
</blockquote> 
<p>A couple of diagrams will make it easier to digest this information. Figure 5 shows the trust configuration. There is a one-way outgoing forest trust from Example.com to Example.local. Example.local has a UPN suffix named ExampleSuffix.local added to it. Example.local also has a child domain named Child and a tree domain named ExampleTree.local. By default, users in Example.local and Child.Example.local will be able to authenticate to resources in Example.com. Users in the ExampleTree.local domain will <em>not</em> be able to authenticate to resources in Example.com, unless the name suffix route for ExampleTree.local is enabled on the trust object in Example.com.<br /> &nbsp;<br /> </p>
<div class="wp-caption aligncenter" id="attachment_22917" style="width: 433px;">
 <img alt="Figure 5: Multi-domain and suffix forest with a trust" class="size-full wp-image-22917" height="243" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2021/11/12/Trusts-AWS-Managed-Microsoft-AD-5.png" width="423" />
 <p class="wp-caption-text" id="caption-attachment-22917">Figure 5: Multi-domain and suffix forest with a trust</p>
</div>
<p></p> 
<p>Figure 6 is from the trust properties dialog from the Example.com forest of a trust between Example.com and Example.local. As you can see, *.example.local is enabled. But the custom UPN suffix ExampleSuffix.local and the tree domain ExampleTree.local are disabled by default.<br /> &nbsp;<br /> </p>
<div class="wp-caption aligncenter" id="attachment_22918" style="width: 410px;">
 <img alt="Figure 6: Example.local trusts details" class="size-full wp-image-22918" height="472" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2021/11/12/Trusts-AWS-Managed-Microsoft-AD-6.png" width="400" />
 <p class="wp-caption-text" id="caption-attachment-22918">Figure 6: Example.local trusts details</p>
</div>
<p></p> 
<h3>Selective authentication</h3> 
<p>With AWS Managed Microsoft AD and self-managed AD, you have the option of configuring Selective Authentication. This option restricts authentication access over a trust to only the users in a trusted domain or forest who have been explicitly given authentication permissions to computer objects that reside in the trusting domain or forest.</p> 
<p>When you use domain or forest-wide authentication, depending on the trust direction, users can authenticate across the trust. Authentication by itself doesn’t provide access—users have to be delegated permissions to access resources. When Selective Authentication is enabled, you must set the <strong>Allowed to Authenticate</strong> permission on each computer object the trusted user will be accessing, in addition to any other permissions that are required to access the computer object.</p> 
<p>While Selective Authentication is a way to provide additional hardening of trusts, it requires a significant amount of planning and delegation, because you have to set the <strong>Allowed to Authenticate</strong> permission on all computer objects that are being accessed. It can also make troubleshooting permissions and trust issues more difficult.</p> 
<p>For more details on Selective Authentication, see <a href="https://docs.microsoft.com/en-us/previous-versions/windows/it-pro/windows-server-2003/cc755321(v=ws.10)?redirectedfrom=MSDN#selective-authentication" rel="noopener noreferrer" target="_blank">Selective Authentication</a> and <a href="https://docs.microsoft.com/en-us/previous-versions/windows/it-pro/windows-server-2008-R2-and-2008/cc816580(v=ws.10)" rel="noopener noreferrer" target="_blank">Configuring Selective Authentication Settings</a> in the Microsoft documentation.</p> 
<h3>SID filtering</h3> 
<p>I won’t spend a lot of time on the subject of SID filtering, since this feature is enabled in AWS Managed Microsoft AD and can’t be disabled. SID filtering prevents malicious users who have domain or enterprise administrator level access in a trusted forest from granting elevated user rights to a trusting forest. It does this by preventing misuse of the attributes containing SIDs on security principals in the trusted forest. For example, a malicious user with administrative credentials located in a trusted forest could, through various means, obtain the SID information of a domain or enterprise admin in the trusting forest. After obtaining the SID of an administrator from the trusting forest, a malicious user with administrative credentials can add that SID to the SID history attribute of a security principal in the trusted forest and attempt to gain full access to the trusting forest and the resources within it.</p> 
<p>Keeping SID filtering disabled on your on-premises domain can open your domain up to risks from malicious users. We understand that during a domain migration, you may need to disable it to allow an object’s SID from the original domain to be used during the migration. But in AWS Managed Microsoft AD, this filtering cannot be disabled. See <a href="https://docs.microsoft.com/en-us/previous-versions/windows/it-pro/windows-server-2003/cc755321(v=ws.10)?redirectedfrom=MSDN#sid-filtering" rel="noopener noreferrer" target="_blank">SID Filtering</a> for more details.</p> 
<h3>Network ports that are required to create trusts</h3> 
<p>The following network ports are required to be open between domain controllers on both domains or forests prior to attempting to create a trust. Note, the Security Group used by your AWS Managed Microsoft AD directory already has these inbound ports open. You will need to adjust the outbound rules of the Security Group to let it communicate with the to be trusted domains or forests. The following table is based on <a href="https://docs.microsoft.com/en-us/troubleshoot/windows-server/identity/config-firewall-for-ad-domains-and-trusts#windows-server-2008-and-later-versions" rel="noopener noreferrer" target="_blank">Microsoft’s recommendations</a>. Depending on your use case, some of these ports might not need to be opened. For example, if LDAP over SSL isn’t configured, then TCP 636 isn’t needed.</p> 
<table border="1" style="margin-left: 45px;" width="0"> 
 <tbody> 
  <tr> 
   <td style="padding-left: 10px;" width="200"><strong>Port</strong></td> 
   <td style="padding-left: 10px;" width="200"><strong>Protocol</strong></td> 
   <td style="padding-left: 10px;" width="300"><strong>Service</strong></td> 
  </tr> 
  <tr style="padding-left: 10px;"> 
   <td style="padding-left: 10px;" width="200">53</td> 
   <td style="padding-left: 10px;" width="200">TCP and UDP</td> 
   <td style="padding-left: 10px;" width="300">DNS</td> 
  </tr> 
  <tr style="padding-left: 10px;"> 
   <td style="padding-left: 10px;" width="200">88</td> 
   <td style="padding-left: 10px;" width="200">TCP and UDP</td> 
   <td style="padding-left: 10px;" width="300">Kerberos</td> 
  </tr> 
  <tr style="padding-left: 10px;"> 
   <td style="padding-left: 10px;" width="200">123</td> 
   <td style="padding-left: 10px;" width="200">UDP</td> 
   <td style="padding-left: 10px;" width="300">Windows Time</td> 
  </tr> 
  <tr style="padding-left: 10px;"> 
   <td style="padding-left: 10px;" width="200">135</td> 
   <td style="padding-left: 10px;" width="200">TCP</td> 
   <td style="padding-left: 10px;" width="300">Remote Procedure Call (RPC)</td> 
  </tr> 
  <tr style="padding-left: 10px;"> 
   <td style="padding-left: 10px;" width="200">389</td> 
   <td style="padding-left: 10px;" width="200">TCP and UDP</td> 
   <td style="padding-left: 10px;" width="300">Lightweight Directory Access Protocol (LDAP)</td> 
  </tr> 
  <tr style="padding-left: 10px;"> 
   <td style="padding-left: 10px;" width="200">445</td> 
   <td style="padding-left: 10px;" width="200">TCP</td> 
   <td style="padding-left: 10px;" width="300">Server Message Block (SMB)</td> 
  </tr> 
  <tr style="padding-left: 10px;"> 
   <td style="padding-left: 10px;" width="200">464</td> 
   <td style="padding-left: 10px;" width="200">TCP and UDP</td> 
   <td style="padding-left: 10px;" width="300">Kerberos Password Change</td> 
  </tr> 
  <tr style="padding-left: 10px;"> 
   <td style="padding-left: 10px;" width="200">636</td> 
   <td style="padding-left: 10px;" width="200">TCP</td> 
   <td style="padding-left: 10px;" width="300">LDAP over SSL</td> 
  </tr> 
  <tr style="padding-left: 10px;"> 
   <td style="padding-left: 10px;" width="200">3268</td> 
   <td style="padding-left: 10px;" width="200">TCP</td> 
   <td style="padding-left: 10px;" width="300">LDAP Global Catalog (GC)</td> 
  </tr> 
  <tr style="padding-left: 10px;"> 
   <td style="padding-left: 10px;" width="200">3269</td> 
   <td style="padding-left: 10px;" width="200">TCP</td> 
   <td style="padding-left: 10px;" width="300">LDAP GC over SSL</td> 
  </tr> 
  <tr style="padding-left: 10px;"> 
   <td style="padding-left: 10px;" width="200">49152–65535</td> 
   <td style="padding-left: 10px;" width="200">TCP and UDP</td> 
   <td style="padding-left: 10px;" width="300">RPC</td> 
  </tr> 
 </tbody> 
</table> 
<p id="_Trust_Creation_Process"></p> 
<h2>Trust creation process overview</h2> 
<p>AWS Managed Microsoft AD is based on Windows Server Active Directory Domain Services, which means that Active Directory trusts function the same way they do with self-managed Active Directory. The only difference is how the trust is created. You use the AWS Management Console or APIs to create the trust for the AWS Managed Microsoft AD side. This process has been documented thoroughly in the <a href="https://docs.aws.amazon.com/directoryservice/latest/admin-guide/ms_ad_tutorial_setup_trust.html" rel="noopener noreferrer" target="_blank">AWS Directory Service Administration Guide</a>, so I won’t go into detail on the steps.</p> 
<p>The high-level overview of the process is:</p> 
<ol> 
 <li>Ensure that network and DNS name resolution is available and functional between the domains.</li> 
 <li>Create the trust on the on-premises Active Directory.</li> 
 <li>Complete the trust on the AWS Managed Microsoft AD in the <a href="https://console.aws.amazon.com/directoryservicev2/" rel="noopener noreferrer" target="_blank">AWS Directory Service console</a>.</li> 
</ol> 
<p id="_Common_trust_scenarios"></p> 
<h2>Common trust scenarios with AWS Managed Microsoft AD</h2> 
<p>When you create trust between an on-premises domain and AWS Managed Microsoft AD, there are some items to take into consideration that will help you decide what direction of trust you need to deploy. In this post, I’ll cover a couple of the most common scenarios.</p> 
<h3>All scenarios: Selecting a trust type</h3> 
<p>Let’s start with the choice between a Forest or External trust. We generally recommend using a Forest trust type. The reason for that is that Forest trusts fully support Kerberos without any caveats. With that said, if you have a specific requirement to implement an External trust, you can do so—just be aware of <a href="https://docs.microsoft.com/en-us/previous-versions/windows/it-pro/windows-server-2003/dd560679(v=ws.10)?redirectedfrom=MSDN#external-trusts-vs-forest-trusts" rel="noopener noreferrer" target="_blank">these caveats</a>.</p> 
<h3>Scenario 1: Use AWS Managed Microsoft AD as a resource forest for Amazon RDS, Amazon FSx for Windows File Server, or Amazon EC2 instances</h3> 
<p>In this scenario, you might want to use AWS Managed Microsoft AD as a resource forest for <a href="https://aws.amazon.com/rds/" rel="noopener noreferrer" target="_blank">Amazon RDS</a>, <a href="https://aws.amazon.com/fsx/windows/" rel="noopener noreferrer" target="_blank">Amazon FSx for Windows File Server</a>, or <a href="https://aws.amazon.com/ec2" rel="noopener noreferrer" target="_blank">Amazon Elastic Compute Cloud (Amazon EC2)</a>. AWS Managed Microsoft AD is going to be a resource domain, and user accounts will reside on the on-premises side of the trust and need to be able to access the resources in the AWS Managed Microsoft AD side of the trust.</p> 
<p>In this scenario, the AWS applications (Amazon RDS, Amazon FSx for Windows File Server, or Amazon EC2) don’t require a two-way trust to function, because they are natively integrated with Active Directory. This tells you that you only need authentication to flow one way. This scenario requires a one-way incoming trust on the on-premises domain and one-way outgoing trusts on the AWS Managed Microsoft AD domain. Figure 7 demonstrates this.<br /> &nbsp;<br /> </p>
<div class="wp-caption aligncenter" id="attachment_22919" style="width: 524px;">
 <img alt="Figure 7: A one-way trust" class="size-full wp-image-22919" height="155" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2021/11/12/Trusts-AWS-Managed-Microsoft-AD-7.png" width="514" />
 <p class="wp-caption-text" id="caption-attachment-22919">Figure 7: A one-way trust</p>
</div>
<p></p> 
<h3>Scenario 2: Use AWS Managed Microsoft AD as a resource forest for all other supported AWS applications</h3> 
<p>In this scenario, you want to use AWS Managed Microsoft AD as a resource domain for all other <a href="https://docs.aws.amazon.com/directoryservice/latest/admin-guide/ms_ad_app_compatibility.html" rel="noopener noreferrer" target="_blank">supported AWS applications</a> that aren’t included in Scenario 1. As the previous scenario stated, AWS Managed Microsoft AD will be a resource domain, and the user accounts will reside on the on-premises side of the trust and need to be able to access the resources in the AWS Managed Microsoft AD.</p> 
<p>In this scenario, AWS applications (<a href="https://aws.amazon.com/chime" rel="noopener noreferrer" target="_blank">Amazon Chime</a>, <a href="https://aws.amazon.com/connect/" rel="noopener noreferrer" target="_blank">Amazon Connect</a>, <a href="https://aws.amazon.com/quicksight/" rel="noopener noreferrer" target="_blank">Amazon QuickSight</a>, <a href="https://aws.amazon.com/single-sign-on/" rel="noopener noreferrer" target="_blank">AWS Single Sign-On</a>, <a href="https://aws.amazon.com/workdocs" rel="noopener noreferrer" target="_blank">Amazon WorkDocs</a>, <a href="https://aws.amazon.com/workmail/" rel="noopener noreferrer" target="_blank">Amazon WorkMail</a>, <a href="https://aws.amazon.com/workspaces" rel="noopener noreferrer" target="_blank">Amazon WorkSpaces</a>, <a href="https://docs.aws.amazon.com/vpn/latest/clientvpn-admin/what-is.html" rel="noopener noreferrer" target="_blank">AWS Client VPN</a>, <a href="https://aws.amazon.com/console/" rel="noopener noreferrer" target="_blank">AWS Management Console</a>, and <a href="https://aws.amazon.com/aws-transfer-family" rel="noopener noreferrer" target="_blank">AWS Transfer Family</a>) need to be able to look up objects from the on-premises domain in order for them to function. This tells you that authentication needs to flow both ways. This scenario requires a two-way trust between the on-premises and AWS Managed Microsoft AD domains. Figure 8 demonstrates this.<br /> &nbsp;<br /> </p>
<div class="wp-caption aligncenter" id="attachment_22920" style="width: 524px;">
 <img alt="Figure 8: A two-way trust" class="size-full wp-image-22920" height="155" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2021/11/12/Trusts-AWS-Managed-Microsoft-AD-8.png" width="514" />
 <p class="wp-caption-text" id="caption-attachment-22920">Figure 8: A two-way trust</p>
</div>
<p></p> 
<p id="_Common_Trusts_Myths"></p> 
<h2>Common trust myths and misconceptions</h2> 
<p>I have had many conversations with customers concerning trusts between their on-premises domain and their AWS Managed Microsoft AD domain. These are some of the common myths and misconceptions we’ve come across in our conversations.</p> 
<p><strong>Trusts synchronize objects between each domain.</strong></p> 
<p>This is false. A trust between domains or forests acts as a bridge that allows validated authentication requests, in the form of Kerberos or NTLM traffic, to travel between domains or forests. Objects are not synchronized between the domains or forests. Only the trust password is synchronized, which is used for Kerberos.</p> 
<p><strong>My password is passed over the trust when authenticating.</strong></p> 
<p>This is false. As I showed earlier in the <a href="#_Starting_with_Kerberos">Starting with Kerberos section</a>, when authenticating across trusts, the user’s password is not passed between domains. The only things passed between domains are the Ticket Granting Service (TGS) requests and responses, which are generated in real time, are single use, and expire within hours.</p> 
<p><strong>A one-way trust allows bidirectional authentication.</strong></p> 
<p>This is false. One-way trusts allow authentications to traverse in one direction only. Users or objects from the trusted domain are able to authenticate and, if they are delegated, to access resources in the trusting domain. Users in the trusting domain can’t authenticate into the trusted domain, and aren’t granted permissions to access resources. Let’s say there is an Amazon FSx file system in Example.local and a one-way trust between Example.com (outgoing trust direction) and Example.local (incoming trust direction). A user in Example.com can’t be delegated permission to the Amazon FSx file system Example.local with the current trust configuration. That’s the nature of a one-way trust.</p> 
<p><strong>Trusts are inherently insecure by default.</strong></p> 
<p>This is false, although an improperly configured trust can increase your risk and exposure. Trusts by themselves do very little to increase an Active Directory’s attack surface. You should always use best practices when creating a trust to minimize risk. For example, a trust without a purpose should be removed. You should disable the SID History unless you’re in the process of migrating domains. See <a href="https://docs.microsoft.com/en-us/previous-versions/windows/it-pro/windows-server-2003/cc755321(v=ws.10)" rel="noopener noreferrer" target="_blank">Security Considerations for Trusts</a> for more guidance on securing trusts.</p> 
<p><strong>Users in the trusted domain are granted permissions to my domain when a trust is created.</strong></p> 
<p>This is false. By default, with two-way trusts, objects have read-only permission to Active Directory in both directions. Objects are not delegated permissions or access to resources or servers by default. For example, if you want a user to log into a computer in another domain, you first must delegate the user access to the resource in the other domain. Without that delegation, the user won’t be able to access the resource.</p> 
<p id="_Troubleshooting_Trusts"></p> 
<h2>Troubleshooting trusts</h2> 
<p>Based on our experience working with many customers, the vast majority of trust configuration issues are either DNS resolution or networking connectivity errors. These are some troubleshooting steps to help you resolve any of these common issues:</p> 
<ul> 
 <li>Check whether you allowed outbound networking traffic on the AWS Managed Microsoft AD. See <a href="https://docs.aws.amazon.com/directoryservice/latest/admin-guide/microsoftadtruststep1.html#configurevpc1" rel="noopener noreferrer" target="_blank">Step 1: Set up your environment for trusts</a> to learn how to find your directory’s security group and how to modify it.</li> 
 <li>If the DNS server or the network for your on-premises domain uses a public (non-RFC 1918) IP address space, follow these steps: 
  <ol> 
   <li>In the <a href="https://console.aws.amazon.com/directoryservicev2/" rel="noopener noreferrer" target="_blank">AWS Directory Service console</a>, go to the <strong>IP routing</strong> section for your directory, choose <strong>Actions</strong>, and then choose <strong>Add route</strong>.</li> 
   <li>Enter the IP address block of your DNS server or on-premises network using CIDR format, for example 203.0.113.0/24. <p>This step isn’t necessary if both your DNS server and your on-premises network are using <a href="https://datatracker.ietf.org/doc/html/rfc1918#section-3" rel="noopener noreferrer" target="_blank">RFC 1918</a> private IP address spaces. </p></li> 
  </ol> </li> 
 <li>After you verify the security group and check whether any applicable routes are required, launch a Windows Server instance and join it to the AWS Managed Microsoft AD directory. See <a href="https://docs.aws.amazon.com/directoryservice/latest/admin-guide/microsoftadbasestep3.html#configureec2" rel="noopener noreferrer" target="_blank">Step 3: Deploy an EC2 instance to manage your AWS Managed Microsoft AD</a> to learn how to do this. Once the instance is launched, do the following: 
  <ul> 
   <li>Run the following PowerShell command to test DNS connectivity:<br /> <code>Resolve-DnsName -Name 'example.local' -DnsOnly</code></li> 
  </ul> </li> 
 <li>You should also look through the message explanations in the <a href="https://docs.aws.amazon.com/directoryservice/latest/admin-guide/ms_ad_troubleshooting_trusts.html" rel="noopener noreferrer" target="_blank">Trust creation status reasons</a> guide in the AWS Directory Service documentation.</li> 
</ul> 
<h2>Summary of AWS Managed Microsoft AD trust considerations</h2> 
<p>In this blog post, I covered Kerberos authentication over Active Directory trusts and provided details on what Active Directory trusts are and how they function. Here’s a quick list of items that you should consider when you plan trust creation with AWS Managed Microsoft AD:</p> 
<ul> 
 <li>Ensure that you have a network connection and the appropriate network ports opened between both domains. Note, it is recommended all Active Directory traffic occur over private network connection like a VPN or Direct Connect.</li> 
 <li>Ensure that DNS resolution is working on both sides of the trust.</li> 
 <li>Decide whether you will implement selective authentication. If it will be used, plan your Active Directory access control list (ACL) delegation strategy before implementation.</li> 
 <li>As of this blog’s publication, keep in mind that AWS Managed Microsoft AD currently supports Forest trusts and External trusts only.</li> 
 <li>Ensure that Kerberos preauthentication is enabled for all objects that traverse trusts with AWS Managed Microsoft AD.</li> 
 <li>If you find that you need a name suffix route enabled for your trust, open a support case with AWS Support, requesting that the name suffix route be enabled.</li> 
 <li>Finally, review <a href="https://docs.microsoft.com/en-us/previous-versions/windows/it-pro/windows-server-2003/cc755321(v=ws.10)?redirectedfrom=MSDN#selective-authentication" rel="noopener noreferrer" target="_blank">Security Considerations for Trusts: Domain and Forest Trusts</a> for additional considerations for trust configuration.</li> 
</ul> 
<p>If you have feedback about this post, submit comments in the <strong>Comments</strong> section below. If you have questions about this post, start a new thread on the <a href="https://forums.aws.amazon.com/forum.jspa?forumID=180**" rel="noopener noreferrer" target="_blank">AWS Directory Service forum</a>.</p> 
<p><strong>Want more AWS Security how-to content, news, and feature announcements? Follow us on <a href="https://twitter.com/AWSsecurityinfo" rel="noopener noreferrer" target="_blank" title="Twitter">Twitter</a>.</strong></p> 
<footer> 
 <div class="blog-author-box"> 
  <div class="blog-author-image"> 
   <img alt="Author" class="alignnone size-full wp-image-17843" height="160" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2020/12/11/Jeremy-Girven-Author.jpg" width="120" /> 
  </div> 
  <h3 class="lb-h4">Jeremy Girven</h3> 
  <p>Jeremy is a solutions architect specializing in Microsoft workloads on AWS. He has over 15 years’ experience with Microsoft Active Directory and over 23 years of industry experience. One of his fun projects is using SSM to automate the Active Directory build processes in AWS. To see more, check out the <a href="https://aws.amazon.com/quickstart/architecture/active-directory-ds/" rel="noopener noreferrer" target="_blank">Active Directory AWS QuickStart</a>.</p> 
  <p></p>
 </div> 
</footer>
