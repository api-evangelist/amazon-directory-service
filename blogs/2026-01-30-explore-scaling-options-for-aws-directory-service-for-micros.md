---
title: "Explore scaling options for AWS Directory Service for Microsoft Active Directory"
url: "https://aws.amazon.com/blogs/security/explore-scaling-options-for-aws-directory-service-for-microsoft-active-directory/"
date: "Fri, 30 Jan 2026 19:51:15 +0000"
author: "Nahuel Benavidez"
feed_url: "https://aws.amazon.com/blogs/security/tag/aws-directory-service/feed/"
---
<div class="Page-articleBody"> 
 <div class="RichTextArticleBody RichTextBody"> 
  <p>You can use <span class="LinkEnhancement"><a class="Link" href="https://aws.amazon.com/directoryservice/features/?nc=sn&amp;loc=2" rel="noopener" target="_blank">AWS Directory Service for Microsoft Active Directory</a></span> as your primary Active Directory Forest for hosting your users’ identities. Your IT teams can continue using existing skills and applications while your organization benefits from the enhanced security, reliability, and scalability of AWS managed services. You can also run AWS Managed Microsoft AD as a <i>resource forest</i>. In this configuration, AWS Managed Microsoft AD serves supported AWS services while users’ identities remain under exclusive control of your organization on a self-managed Active Directory. As your organization grows and scales, so will your AWS Managed Microsoft AD deployments.</p> 
  <p>In this post, you’ll learn how to use <span class="LinkEnhancement"><a class="Link" href="https://aws.amazon.com/cloudwatch" rel="noopener" target="_blank">Amazon CloudWatch</a></span> dashboards to monitor key performance metrics of your AWS Managed Microsoft AD deployment to track and analyze a directory’s performance over time. You can then use that information to determine when and how best to scale directory services for optimal performance.</p> 
  <div class="RichTextHeading"> 
   <h2>Scaling your Active Directory</h2> 
  </div> 
  <p>When you deploy AWS Managed Microsoft AD, the service initially creates two domain controller instances in two separate subnets of the same <span class="LinkEnhancement"><a class="Link" href="https://docs.aws.amazon.com/vpc/latest/userguide/what-is-amazon-vpc.html" rel="noopener" target="_blank">virtual private cloud (VPC)</a></span>. This architecture economically provides resiliency and high availability with a minimal set of resources. This initial configuration enables every feature that AWS Managed Microsoft AD offers. As your organization grows, its workflows will become larger and more complex, requiring that you scale your directories accordingly. AWS Managed Microsoft AD simplifies and makes the scaling process secure with minimal administrative effort. When it’s time to scale a directory, AWS Managed Microsoft AD offers two options: scale-up or scale-out.</p> 
  <div class="RichTextHeading"> 
   <h3>Understanding scale-up and scale-out</h3> 
  </div> 
  <p>Scale-up—also called <span class="LinkEnhancement"><a class="Link" href="https://docs.aws.amazon.com/directoryservice/latest/admin-guide/ms_ad_upgrade_edition.html" rel="noopener" target="_blank">upgrading your AWS Managed Microsoft AD</a></span>—means changing the edition of an AWS Managed Microsoft AD from Standard to Enterprise. Enterprise Edition delivers larger domain controller instances, with higher compute capacity and larger storage for Active Directory objects. When a directory scales up, it retains the same number of domain controller instances that it previously had with larger quotas. Instances are replaced one at a time to minimize disruptions to production workflows.</p> 
  <p>A few features offered by the service are a better fit for the size and compute power of Enterprise Edition AWS Managed Microsoft AD and so are only available in Enterprise Edition. Consider scaling-up your directory if you encounter any of the following scenarios:</p> 
  <ul class="rte2-style-ul" id="rte-8842b010-f8bb-11f0-9db3-f596f7553a28"> 
   <li>You plan to replicate your directory across multiple AWS Regions. Multi-Region replication is only available in Enterprise Edition.</li> 
   <li>The number of Active Directory objects in the directory will exceed the recommended threshold of 30,000 objects for Standard Edition. Enterprise Edition can accommodate up to 500,000 directory objects.</li> 
   <li>You plan to share your directory with more than 25 other AWS accounts. The default directory sharing quota is 25 accounts for Standard Edition and 500 for Enterprise Edition.</li> 
  </ul> 
  <p><b>Important: </b>Scaling up a directory from Standard to Enterprise is a one-way operation that cannot be reverted and operates at a higher hourly price.</p> 
  <p>Scale-out means <span class="LinkEnhancement"><a class="Link" href="https://docs.aws.amazon.com/directoryservice/latest/admin-guide/ms_ad_deploy_additional_dcs.html" rel="noopener" target="_blank">deploying additional domain controllers for your AWS Managed Microsoft AD</a></span>. You can scale out both Standard or Enterprise directories and can scale out different Regions independently. You don’t need to scale every Region to the same number of domain controller instances. When scale-out takes place, additional domain controller instances with the same compute resources and storage capacity as existing ones are launched in the same subnets.</p> 
  <p>Because some operations cannot be reverted, it’s important to understand the impact of each scaling operation. It’s preferable to scale out the number of domain controllers first, because you can revert that change if necessary. Consider scaling up first only if you need a feature that’s only available in Enterprise Edition.</p> 
  <div class="RichTextHeading"> 
   <h3>Making an informed decision using CloudWatch</h3> 
  </div> 
  <p>Since December 2021, <span class="LinkEnhancement"><a class="Link" href="https://aws.amazon.com/about-aws/whats-new/2021/12/aws-managed-microsoft-ad-amazon-cloudwatch/" rel="noopener" target="_blank">AWS Managed Microsoft AD helps optimize scaling decisions with directory metrics in Amazon CloudWatch</a></span>. <span class="LinkEnhancement"><a class="Link" href="https://aws.amazon.com/cloudwatch/" rel="noopener" target="_blank">Amazon CloudWatch</a></span> metrics are a time-ordered set of data-points about performance indicators of a system that you can use to monitor and analyze performance over time. Metrics are stored as a time-series set and each data point has an associated timestamp. By using CloudWatch, you can create alarms based on metrics and visualize and analyze metrics to derive new insights.</p> 
  <p>To understand the performance of a directory over time, define the key performance metrics based on your workload when you create the directory. Record the initial values of those metrics to create a performance baseline. Periodically revisit and compare data points for the same metrics to understand trends and use of resources over time. Based on the information provided by the performance baseline and periodic follow-ups, you can decide when to scale your directory and what scaling method to use. This process is depicted in Figure 1.</p> 
  <div class="wp-caption alignnone" id="attachment_41316" style="width: 895px;">
   <img alt="Figure 1: Decision-making process for scaling an Active Directory implementation" class="wp-image-41316 size-full" height="798" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2026/01/26/3119.1.png" width="885" />
   <p class="wp-caption-text" id="caption-attachment-41316">Figure 1: Decision-making process for scaling an Active Directory implementation</p>
  </div> 
  <p>Depending on the characteristics of your workload, you might face different resource constraints in your directory system. From an infrastructure perspective, the more commonly demanded resources are:</p> 
  <ul class="rte2-style-ul" id="rte-e5c0f904-f8ba-11f0-9177-5514d19b2f66"> 
   <li>Network Interface: Current Bandwidth</li> 
   <li>Processor: % Processor Time</li> 
   <li>LogicalDisk: % Free Space</li> 
  </ul> 
  <p>From an Active Directory perspective, consider metrics such as:</p> 
  <ul class="rte2-style-ul" id="rte-e5c0f905-f8ba-11f0-9177-5514d19b2f66"> 
   <li>NTDS: LDAP Searches/sec</li> 
   <li>NTDS: ATQ Estimated Queue Delay</li> 
  </ul> 
  <p>The following table is an example decision matrix based on which resource is constrained.</p> 
  <table border="1px" cellpadding="10px" style="border-collapse: separate; text-indent: initial; border-spacing: 2px; border-color: gray; width: 100%;"> 
   <tbody> 
    <tr> 
     <td style="padding: 10px; border: 1px solid #dddddd;"><b>Constrained resource</b></td> 
     <td style="padding: 10px; border: 1px solid #dddddd;"><b>Recommended action</b></td> 
    </tr> 
    <tr> 
     <td style="padding: 10px; border: 1px solid #dddddd;">% Processor Time</td> 
     <td style="padding: 10px; border: 1px solid #dddddd;">Scale out</td> 
    </tr> 
    <tr> 
     <td style="padding: 10px; border: 1px solid #dddddd;">I/O Database Reads Average Latency</td> 
     <td style="padding: 10px; border: 1px solid #dddddd;">Scale out</td> 
    </tr> 
    <tr> 
     <td style="padding: 10px; border: 1px solid #dddddd;">Committed Bytes in Use</td> 
     <td style="padding: 10px; border: 1px solid #dddddd;">Scale out</td> 
    </tr> 
    <tr> 
     <td style="padding: 10px; border: 1px solid #dddddd;">% Free Space</td> 
     <td style="padding: 10px; border: 1px solid #dddddd;">Scale up</td> 
    </tr> 
   </tbody> 
  </table> 
  <p>For example, you can create a CloudWatch alarm that will trigger when <i>Processor: % Processor Time</i> is over 80% for more than 5 minutes. If this alarm triggers often, it could be a signal that domain controller instances are struggling to service the regular volume of user authentication requests. In such a scenario, you might consider scaling-out an additional domain controller to guarantee the service’s SLA. Conversely, if the <i>LogicalDisk: % Free Space</i> drops below 10% and trends downwards, you might consider scaling-up to Enterprise Edition, because it provides a larger capacity for directory objects.</p> 
  <p>To facilitate tracking and analyzing performance of AWS Managed Microsoft AD over time, you can use Amazon CloudWatch to create a custom dashboard including relevant metrics.</p> 
  <div class="RichTextHeading"> 
   <h2>Prerequisites</h2> 
  </div> 
  <p>Before you get started, make sure that you have the following prerequisites in place:</p> 
  <ul class="rte2-style-ul" id="rte-8842d720-f8bb-11f0-9db3-f596f7553a28"> 
   <li>An <span class="LinkEnhancement"><a class="Link" href="https://docs.aws.amazon.com/accounts/latest/reference/accounts-welcome.html" rel="noopener" target="_blank">AWS account</a></span></li> 
   <li>An <span class="LinkEnhancement"><a class="Link" href="https://docs.aws.amazon.com/IAM/latest/UserGuide/introduction.html" rel="noopener" target="_blank">AWS Identity and Access Management (IAM)</a></span> user or role with permissions to perform <span class="LinkEnhancement"><a class="Link" href="https://aws.amazon.com/directoryservice" rel="noopener" target="_blank">AWS Directory Service</a></span> operations and CloudWatch operations</li> 
   <li>An <span class="LinkEnhancement"><a class="Link" href="https://docs.aws.amazon.com/vpc/latest/userguide/what-is-amazon-vpc.html" rel="noopener" target="_blank">Amazon Virtual Private Cloud (Amazon VPC)</a></span> VPC configured in each Region</li> 
   <li>At least two <span class="LinkEnhancement"><a class="Link" href="https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Scenario2.html" rel="noopener" target="_blank">private subnets</a></span> in the VPC</li> 
   <li>An <span class="LinkEnhancement"><a class="Link" href="https://docs.aws.amazon.com/directoryservice/latest/admin-guide/ms_ad_getting_started.html" rel="noopener" target="_blank">AWS Managed Microsoft AD</a></span> directory</li> 
  </ul> 
  <div class="RichTextHeading"> 
   <h2>Create a CloudWatch dashboard</h2> 
  </div> 
  <p>With the prerequisites in place, you’re ready to create a CloudWatch dashboard to track directory service metrics. For more information, see <span class="LinkEnhancement"><a class="Link" href="https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/GettingStarted.html" rel="noopener" target="_blank">Getting started with CloudWatch automatic dashboards</a></span>.</p> 
  <p><b>To create a dashboard:</b></p> 
  <ol class="rte2-style-ol" id="rte-8842d721-f8bb-11f0-9db3-f596f7553a28" start="1"> 
   <li>Open the <span class="LinkEnhancement"><a class="Link" href="https://console.aws.amazon.com/cloudwatch/" rel="noopener" target="_blank">AWS Management Console for CloudWatch</a></span>.</li> 
   <li>In the navigation pane, choose <b>Dashboards</b>, and then choose <b>Create dashboard</b>.</li> 
   <li>In the <b>Create new dashboard</b> dialog box, enter a name for the dashboard and then choose <b>Create dashboard</b>.</li> 
   <li>When the <b>Add widget </b>window appears: 
    <ol class="rte2-style-ol" id="rte-8842d722-f8bb-11f0-9db3-f596f7553a28" start="1" type="a"> 
     <li>Under <b>Data sources types</b>, select <b>CloudWatch</b>.</li> 
     <li>Under <b>Data type</b>, select <b>Metrics</b>.</li> 
     <li>Under <b>Widget type</b>, select <b>Line</b>.</li> 
     <li>Choose <b>Next</b>.</li> 
    </ol> </li> 
   <li>In the <b>Add metric graph </b>window, choose <b>DirectoryService</b> and then select <b>Processor</b> as the <b>Metric category </b>and<b> % Processor Time </b>under <b>Metric name</b>. Select each instance of the metric, represented as the<b><i> </i>Domain Controller IP</b>, for one <b>Directory ID</b>.</li> 
   <li>Choose <b>Create widget</b>.<br /> 
    <blockquote>
     <p>Note: if there are multiple directories in the same Region, all instances (domain controllers IPs) will be available for selection. To help ensure effective monitoring and alarms, create a separate dashboard for each directory.</p>
    </blockquote> </li> 
   <li>Choose the plus sign (<b>+</b>) at the top of the window to add more widgets. Repeat steps 1–6 to add additional widgets for other relevant metrics. In this example the metric categories and names added are: 
    <ul class="rte2-style-ul" id="rte-8842fe31-f8bb-11f0-9db3-f596f7553a28"> 
     <li>Processor: % Processor Time</li> 
     <li>LogicalDisk: % Free Space</li> 
     <li>Memory: Committed Bytes in Use</li> 
     <li>Database: I/O Database Reads Average Latency</li> 
     <li>Network Interface: Current Bandwidth</li> 
     <li>DNS: Recursive Queries/Sec</li> 
    </ul> </li> 
   <li>After adding the desired metrics, choose <b>Save</b>.</li> 
  </ol> 
  <div class="wp-caption alignnone" id="attachment_41323" style="width: 2072px;">
   <img alt="Figure 2: CloudWatch dashboard showing directory services metrics" class="wp-image-41323 size-full" height="1744" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2026/01/27/3119.2.2.png" width="2062" />
   <p class="wp-caption-text" id="caption-attachment-41323">Figure 2: CloudWatch dashboard showing directory services metrics</p>
  </div> 
  <div class="RichTextHeading"> 
   <h2>(Optional) Create an alarm in CloudWatch</h2> 
  </div> 
  <p>Now that you have a dashboard where you can view metrics, consider setting up CloudWatch alarms to alert you when a metric reaches or goes beyond a specified threshold. For more information, see <span class="LinkEnhancement"><a class="Link" href="https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/ConsoleAlarms.html" rel="noopener" target="_blank">Create a CloudWatch alarm based on a static threshold</a></span> and <span class="LinkEnhancement"><a class="Link" href="https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/add_alarm_dashboard.html" rel="noopener" target="_blank">Adding an alarm to a CloudWatch dashboard</a></span>.</p> 
  <p>The following are recommended thresholds to monitor when determining the need to scale an AWS Managed Microsoft AD. These are general recommendations based on standard use cases. You might have to adjust these thresholds to make the best scaling decisions for your organization.</p> 
  <ul class="rte2-style-ul" id="rte-e5c16e32-f8ba-11f0-9177-5514d19b2f66"> 
   <li><b>Processor: % Processor Time:</b> Monitor CPU utilization to understand computational demands on your domain controllers. Set CloudWatch alarms at 80% for a period of 5 minutes. Sustained high values indicate potential sizing issues that might require scaling out your directory.</li> 
   <li><b>LogicalDisk: % Free Space:</b> Maintain at least 25% free space on volumes containing Active Directory data for optimal performance. Set CloudWatch alarms to trigger when free space drops below 20%. Low disk space can severely impact directory operations and require implementing cleanup procedures or scaling up the directory.</li> 
   <li><b>Network Interface: Current Bandwidth:</b> Average network utilization should be kept below 50% of available bandwidth during peak operations for optimal directory responsiveness. Set CloudWatch alarms at 70% utilization to allow room for spikes in activity. Consistently high values suggest network constraints that might require scaling out your directory.</li> 
   <li><b>Memory: Committed Bytes in Use:</b> Monitor memory commitment levels to help ensure that your domain controllers have sufficient memory resources for Active Directory operations. This metric tracks the amount of virtual memory that has been committed, indicating the total memory load on your domain controllers. Set CloudWatch alarms at 80% of the commit limit. Sustained high values can lead to excessive paging, significantly degrading directory performance and potentially causing authentication delays.</li> 
   <li><b>Database: I/O Database Reads Average Latency:</b> Maintain average read latencies below 25 milliseconds. Set CloudWatch alarms at a threshold of 50 milliseconds. If read latencies are consistently elevated, consider scaling-out your directory.</li> 
   <li><b>DNS: Recursive Queries/sec:</b> Given the tight integration of Active Directory with DNS, monitor this metric for stability and predictable patterns. Use CloudWatch anomaly detection rather than fixed thresholds to identify unexpected behaviors that could indicate DNS configuration issues or potential security concerns.</li> 
  </ul> 
  <div class="RichTextHeading"> 
   <h2>Post-scaling considerations</h2> 
  </div> 
  <p>Different resources across your architecture might contain references to the IP addresses of the AWS Managed Microsoft AD. After a scale-out operation that deploys additional domain controller instances on a directory, update existing references to maintain full functionality of workloads. References for the directory’s IP addresses can be found (but might not be limited to) the following services:</p> 
  <ul class="rte2-style-ul" id="rte-e5c1bc50-f8ba-11f0-9177-5514d19b2f66"> 
   <li>Firewall rules</li> 
   <li><span class="LinkEnhancement"><a class="Link" href="https://docs.aws.amazon.com/vpc/latest/userguide/working-with-security-group-rules.html" rel="noopener" target="_blank">Amazon Virtual Private Cloud (Amazon VPC) security groups</a></span></li> 
   <li><span class="LinkEnhancement"><a class="Link" href="https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/resolver-rules-managing.html" rel="noopener" target="_blank">Amazon Route 53 Resolver endpoint rules</a></span></li> 
   <li><span class="LinkEnhancement"><a class="Link" href="https://docs.aws.amazon.com/managedservices/latest/onboardingguide/configure-conditional-forwarder.html" rel="noopener" target="_blank">DNS conditional forwards</a></span></li> 
   <li><span class="LinkEnhancement"><a class="Link" href="https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/create_dashboard.html" rel="noopener" target="_blank">CloudWatch dashboards</a></span></li> 
  </ul> 
  <p>To maintain the full functionality of your workloads after a directory scaling operation, update the following:</p> 
  <ul class="rte2-style-ul" id="rte-e5c1bc55-f8ba-11f0-9177-5514d19b2f66"> 
   <li>Firewall rules that allow traffic to and from the IP addresses of domain controller instances</li> 
   <li>Route53 Resolver endpoint rules and DNS conditional forwarders that forward queries to the directory instances</li> 
   <li>CloudWatch dashboards that display metric data about the directory to include dimensions for the new IP addresses</li> 
  </ul> 
  <div class="RichTextHeading"> 
   <h2>Clean up resources</h2> 
  </div> 
  <p>In this post, you created components that generate costs. Clean up these resources when no longer required to avoid additional charges.</p> 
  <ul id="rte-bde13240-faf1-11f0-853d-c34e46cec0dc"> 
   <li>Remove added domain controller’s IP addresses from firewall rules, resolver endpoint rules and DNS conditional forwarders.</li> 
   <li>Delete the custom CloudWatch dashboards you don’t plan to keep.</li> 
   <li>Scale back existing directories to the previous number of domain controller instances.</li> 
  </ul> 
  <div class="RichTextHeading"> 
   <h2>Conclusion</h2> 
  </div> 
  <p>In this post, you learned how to monitor directory performance metrics using Amazon CloudWatch. By combining performance baselines, monitoring, and planning, you can make informed decisions about when and how to scale a directory safely and efficiently. By scaling directories in a timely manner, you can optimize efficiency and reduce the risk of outages by having a right-sized directory service to support your organization’s workloads.</p> 
  <p>Scale out your directory when your Active Directory-aware workflows have grown over time and the solution requires additional domain controller instances to maintain the service SLA. Scale up your directory when you require a feature that’s only available in Enterprise Edition AWS Managed Microsoft AD, such as multi-Region replication or additional storage to accommodate Active Directory objects. By using the flexible scaling capabilities and independent Regional expansion, you can optimize costs while maintaining appropriate service levels.</p> 
  <p>To learn more about AWS Managed Microsoft AD optimization and monitoring with Amazon CloudWatch, see:</p> 
  <ul id="rte-bde13241-faf1-11f0-853d-c34e46cec0dc"> 
   <li><span class="LinkEnhancement"><a class="Link" href="https://docs.aws.amazon.com/directoryservice/latest/admin-guide/ms_ad_best_practices.html" rel="noopener" target="_blank">AWS Managed Microsoft AD best practices</a></span></li> 
   <li><span class="LinkEnhancement"><a class="Link" href="https://docs.aws.amazon.com/directoryservice/latest/admin-guide/ms_ad_monitor_dc_performance.html#scaledcs" rel="noopener" target="_blank">Determining when to add domain controllers with CloudWatch metrics</a></span></li> 
  </ul> 
  <footer> 
   <div class="blog-author-box">
    <img alt="Nahuel Benavidez" class="alignleft size-full" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2026/01/26/nahuelbenavidez.nahuelb.png" style="width: 93.750px; height: 125px; margin: 12px 18px 6px 12px;" />
    <span class="lb-h4" style="line-height: 2.1em; padding-top: 12px; margin-top: 24px;">Nahuel Benavidez</span>
    <br /> Nahuel is a Sr. CSE in AWS, specializing in AWS Directory Service, Microsoft Technologies, and SQL Server. He enjoys teaming with customers to discover exciting ways to explore AWS services. Nahuel loves to spoil his niece and goddaughters above all else. Also, Dungeons and Dragons (before it was popular), CrossFit, hiking, trekking and, sharing a pint with friends but 
    <i>“just one.”</i>
   </div> 
  </footer> 
 </div> 
</div>
