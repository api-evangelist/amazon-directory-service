---
title: "How to monitor and track failed logins for your AWS Managed Microsoft AD"
url: "https://aws.amazon.com/blogs/security/how-to-monitor-and-track-failed-logins-for-your-aws-managed-microsoft-ad/"
date: "Fri, 02 Jul 2021 17:47:59 +0000"
author: "Tekena Orugbani"
feed_url: "https://aws.amazon.com/blogs/security/tag/aws-directory-service/feed/"
---
<p><a href="https://aws.amazon.com/directoryservice/" rel="noopener noreferrer" target="_blank">AWS Directory Service for Microsoft Active Directory</a> provides customers with the ability to <a href="https://docs.aws.amazon.com/directoryservice/latest/admin-guide/ms_ad_directory_logs.html" rel="noopener noreferrer" target="_blank">review security logs</a> on their AWS Managed Microsoft AD domain controllers by either using a <a href="https://docs.aws.amazon.com/directoryservice/latest/admin-guide/ms_ad_manage_users_groups.html" rel="noopener noreferrer" target="_blank">domain management Amazon Elastic Compute Cloud (Amazon EC2) instance</a> or by <a href="https://docs.aws.amazon.com/directoryservice/latest/admin-guide/ms_ad_enable_log_forwarding.html" rel="noopener noreferrer" target="_blank">forwarding</a> domain controller security event logs to <a href="http://aws.amazon.com/cloudwatch" rel="noopener noreferrer" target="_blank">Amazon CloudWatch Logs</a>.</p> 
<p>You can further improve visibility by monitoring Windows login activities on your AWS Managed Microsoft AD domain-joined EC2 instances, and in this blog post, I show you how. Monitoring and tracking Windows security events on your AWS Managed Microsoft AD domain-joined instances can reveal unexpected activities on your domain-joined EC2 instances so that you can take proactive remediating action.</p> 
<p>For example, every time there is an unsuccessful attempt to log in to a domain-joined EC2 instance or on-premises server by using an AWS Managed Microsoft AD user or a local account, an “Audit Failure” Windows security <a href="https://docs.microsoft.com/en-us/windows/security/threat-protection/auditing/event-4625" rel="noopener noreferrer" target="_blank">event with ID 4625</a> is recorded on the EC2 instance itself. The event data includes details of the account name, workstation name, and source network address. Unsuccessful attempts to log in to non–domain-joined EC2 instances and servers are handled the same way. You can track and monitor these events on an ongoing basis across your fleet of Windows EC2 instances by using the solution described here.</p> 
<h2>Solution overview</h2> 
<p>Figure 1 shows the workflow for the solution.<br /> &nbsp;<br /> </p>
<div class="wp-caption aligncenter" id="attachment_20974" style="width: 1077px;">
 <img alt="Figure 1: Solution architecture" class="size-full wp-image-20974" height="434" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2021/06/26/Monitor-track-failed-logins-1.png" width="1067" />
 <p class="wp-caption-text" id="caption-attachment-20974">Figure 1: Solution architecture</p>
</div>
<p></p> 
<p>The workflow steps are as follows:</p> 
<ol> 
 <li>An <a href="https://aws.amazon.com/cloudwatch/" rel="noopener noreferrer" target="_blank">Amazon CloudWatch</a> agent that is running on the EC2 instances sends the Windows security event logs to Amazon CloudWatch.</li> 
 <li>CloudWatch filters the logs based on the filter you specify. When the configured threshold is met, CloudWatch posts an alert to an SNS topic.</li> 
 <li><a href="https://aws.amazon.com/sns/" rel="noopener noreferrer" target="_blank">Amazon Simple Notification Service (Amazon SNS)</a> invokes an <a href="https://aws.amazon.com/lambda" rel="noopener noreferrer" target="_blank">AWS Lambda</a> function.</li> 
 <li>The Lambda function scans through the events and determines which EC2 instance(s) generated the security events at a frequency that satisfies the configured threshold. It discards any other instances listed in the events that don’t meet the specified criteria. The function sends an email to the configured email address with a high-level description of the event logs and the instance(s) that generated them.</li> 
 <li><a href="https://aws.amazon.com/ses/" rel="noopener noreferrer" target="_blank">Amazon Simple Email Service (Amazon SES</a>) delivers the emails in the specified mailbox.</li> 
</ol> 
<blockquote>
 <p><strong>Note</strong>: Although this example uses email notification via Amazon SES to monitor failed logins, there are opportunities to extend the solution. For example, you can integrate with a Security Information and Event Management (SIEM) tool that may potentially be integrated with a ticketing service and/or some automation or incident response process when a set threshold for failed logins is breached.</p>
</blockquote> 
<p id="Prerequisites"> </p>
<h2>Prerequisites</h2> 
<p></p> 
<p>Before you deploy the solution, you must complete the following steps:</p> 
<ol> 
 <li><a href="https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/create-iam-roles-for-cloudwatch-agent.html" rel="noopener noreferrer" target="_blank">Create AWS Identity and Access Management (IAM) roles</a> for use with the CloudWatch agent</li> 
 <li><a href="https://aws.amazon.com/ses/" rel="noopener noreferrer" target="_blank">Sign up for Amazon SES</a></li> 
 <li><a href="https://docs.aws.amazon.com/ses/latest/DeveloperGuide/verify-email-addresses-procedure.html#verify-email-addresses-procedure-console" rel="noopener noreferrer" target="_blank">Verify the sender and recipient email addresses</a> that you’ll use to send and receive email notifications</li> 
</ol> 
<h2>Deploy the solution</h2> 
<p>The solution I present here involves four main steps:</p> 
<ol> 
 <li>Install and configure the CloudWatch agent for your EC2 instances.</li> 
 <li>Create a metric filter in CloudWatch.</li> 
 <li>Create a CloudWatch alarm based on the metric filter and add SNS notification.</li> 
 <li>Create a Lambda function and subscribe the function to the SNS topic.</li> 
</ol> 
<p id="Step-1"> </p>
<h3>Step 1: Install and configure the CloudWatch agent for all your EC2 instances</h3> 
<p></p> 
<p>The first step is to create an <a href="https://aws.amazon.com/ec2/systems-manager" rel="noopener noreferrer" target="_blank">AWS Systems Manager</a> parameter to contain the JSON configuration for the CloudWatch agent that runs on the EC2 instances. You’ll then use Systems Manager Run Command to install the CloudWatch agent on the instances and to apply the configuration in the Parameter Store to the CloudWatch agent.</p> 
<h4>To install and configure the CloudWatch agent</h4> 
<ol> 
 <li>Open the <a href="http://console.aws.amazon.com/systems-manager" rel="noopener noreferrer" target="_blank">AWS Systems Manager</a> console and in the navigation pane, choose <strong>Parameter Store </strong>to create a new Systems Manager parameter.</li> 
 <li>Give your parameter a name. In my example, I named my parameter <span style="font-family: courier;">AmazonCloudWatch-Windows</span>.</li> 
 <li>For <strong>Tier</strong>, choose <strong>Standard</strong>. For <strong>Type</strong>, choose <strong>String</strong>. For <strong>Data type</strong>, choose <strong>Text</strong>.</li> 
 <li>For the value of the parameter, enter the following JSON configuration and choose <strong>Create Parameter</strong>.<br /> 
  <blockquote>
   <p><strong>Note</strong>: This JSON configuration creates a log group in CloudWatch with the name <span style="font-family: courier;">/aws/SecurityAuditLogs</span>. If you would prefer to use another log group name, you can modify the JSON configuration. Also, if you already have a Systems Manager parameter named <span style="font-family: courier;">AmazonCloudWatch-Windows</span>, you can use any other name of your choice.</p>
  </blockquote> 
  <div class="hide-language"> 
   <pre><code class="lang-text">{ "logs": { "logs_collected": { "windows_events": { "collect_list": [ { "event_format": "xml", "event_levels": [ "VERBOSE", "INFORMATION", "WARNING", "ERROR", "CRITICAL" ], "event_name": "Security", "log_group_name": "/aws/SecurityAuditLogs", "log_stream_name": "{instance_id}" } ] } } }, "metrics": { "metrics_collected": { "statsd": { "metrics_aggregation_interval": 60, "metrics_collection_interval": 10, "service_address": ":8125" } } } }
</code></pre> 
  </div> <p>The <strong>Parameter details</strong> page should look similar to the following.<br /> &nbsp;<br /> </p>
  <div class="wp-caption aligncenter" id="attachment_20975" style="width: 911px;">
   <img alt="Figure 2: Create the System Manager parameter for the CloudWatch agent" class="size-full wp-image-20975" height="810" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2021/06/26/Monitor-track-failed-logins-2.png" width="901" />
   <p class="wp-caption-text" id="caption-attachment-20975">Figure 2: Create the System Manager parameter for the CloudWatch agent</p>
  </div> </li> 
 <li>Next, you’ll use Run Command to install and configure the CloudWatch agent. In the navigation pane, choose <strong>Run Command</strong>.</li> 
 <li>On the <strong>Run a command</strong> page, in the search box, enter <span style="font-family: courier;">Document name prefix: Equals: AWS-ConfigureAWSPackage</span>. Press <strong>Enter</strong> and select the document that appears.</li> 
 <li>Under <strong>Command parameters</strong>, for <strong>Name</strong>, enter <span style="font-family: courier;">AmazonCloudWatchAgent</span>.<br /> &nbsp;<br /> 
  <div class="wp-caption aligncenter" id="attachment_20976" style="width: 954px;">
   <img alt="Figure 3: Install the CloudWatch agent on the instances" class="size-full wp-image-20976" height="873" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2021/06/26/Monitor-track-failed-logins-3.png" width="944" />
   <p class="wp-caption-text" id="caption-attachment-20976">Figure 3: Install the CloudWatch agent on the instances</p>
  </div> </li> 
 <li>Under <strong>Targets, </strong>specify your EC2 instances based on their tags, or choose them manually, and then choose <strong>Run</strong>.</li> 
 <li>To configure the CloudWatch agent, choose <strong>Run Command</strong> again. On the <strong>Run a command</strong> screen, enter <span style="font-family: courier;">Document name prefix: Equals: AmazonCloudWatch-ManageAgent</span>. Press <strong>Enter</strong> and select the document that appears.</li> 
 <li>Under <strong>Command parameters</strong>, for <strong>Optional Configuration Location</strong>, enter the name of the Systems Manager parameter you created earlier. In my example, I used the name <span style="font-family: courier;">AmazonCloudWatch-Windows</span>. Keep the defaults for the other settings.<br /> &nbsp;<br /> 
  <div class="wp-caption aligncenter" id="attachment_20977" style="width: 950px;">
   <img alt="Figure 4: Configure the CloudWatch agent on the instances" class="size-full wp-image-20977" height="872" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2021/06/26/Monitor-track-failed-logins-4.png" width="940" />
   <p class="wp-caption-text" id="caption-attachment-20977">Figure 4: Configure the CloudWatch agent on the instances</p>
  </div> </li> 
 <li>Under <strong>Targets, </strong>specify your EC2 instances based on their tags, or choose them manually, and then choose <strong>Run</strong>.</li> 
</ol> 
<h3>Step 2: Create a metric filter in CloudWatch</h3> 
<p>After the completion of the tasks in Step 1, your EC2 instances should now be sending logs to a log group in <a href="https://aws.amazon.com/cloudwatch/" rel="noopener noreferrer" target="_blank">Amazon CloudWatch</a> called <span style="font-family: courier;">/aws/SecurityAuditLogs</span>. The log group should have log streams named after the EC2 instances that are sending the logs to CloudWatch. The next step is to create a metric filter to filter the noise from the logs.</p> 
<h4>To create a metric filter</h4> 
<ol> 
 <li>Open the <a href="http://console.aws.amazon.com/cloudwatch" rel="noopener noreferrer" target="_blank">CloudWatch console</a> and in the left navigation menu, choose <strong>Log Groups</strong>.</li> 
 <li>Select the check box next to the <span style="font-family: courier;">/aws/SecurityAuditLogs</span> log group, choose <strong>Actions</strong>, and then choose <strong>Create metric filter</strong>.</li> 
 <li>On the<strong> Define pattern </strong>page, enter <span style="font-family: courier;">Audit Failure</span>, keep the defaults for the other settings, and then choose <strong>Next</strong>.</li> 
 <li>Enter values for <strong>Filter name</strong>, <strong>Metric namespace</strong>, <strong>Metric name</strong>, and <strong>Metric value</strong>, and then choose <strong>Next</strong> to create the metric filter.</li> 
</ol> 
<p>&nbsp;<br /> </p>
<div class="wp-caption aligncenter" id="attachment_20978" style="width: 646px;">
 <img alt="Figure 5: Create a CloudWatch metric filter" class="size-full wp-image-20978" height="735" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2021/06/26/Monitor-track-failed-logins-5.png" width="636" />
 <p class="wp-caption-text" id="caption-attachment-20978">Figure 5: Create a CloudWatch metric filter</p>
</div>
<p></p> 
<h3>Step 3: Create a CloudWatch alarm based on the metric filter and add SNS notification</h3> 
<p>In this step, you set a threshold for how many “Audit Failure” events you want to allow within a period of time before triggering an alarm.</p> 
<h4>To create the CloudWatch alarm and add SNS notification</h4> 
<ol> 
 <li>Open the <a href="http://console.aws.amazon.com/sns" rel="noopener noreferrer" target="_blank">Amazon Simple Notification Service</a> console and in the left navigation menu, choose <strong>Topics</strong>.</li> 
 <li>Choose <strong>Create topic</strong>, and then choose <strong>Standard</strong>.</li> 
 <li>Provide a name for your topic, and then choose <strong>Create topic</strong>. In my example, I named the topic <span style="font-family: courier;">WindowsSecurityLogsAlarmNotifications</span>.</li> 
 <li>Open the <a href="http://console.aws.amazon.com/cloudwatch" rel="noopener noreferrer" target="_blank">CloudWatch console</a>, choose <strong>Log groups</strong>, and select the <span style="font-family: courier;">/aws/SecurityAuditLogs</span> log group.</li> 
 <li>Choose the <strong>Metric filters</strong> tab, select the check box next to the <span style="font-family: courier;">WindowsSecurityAuditFailures</span> filter you just created, and choose <strong>Create alarm</strong>.</li> 
 <li>On the <strong>Specify metric and conditions</strong> page, set the parameters as follows: 
  <ol type="a"> 
   <li>For<strong> Statistic</strong>, choose <strong>Sample count</strong>.</li> 
   <li>For<strong> Period</strong>, choose <strong>5 minutes</strong>.</li> 
   <li>For <strong>Threshold type</strong>, choose <strong>Static</strong>.</li> 
   <li>For <strong>Define the alarm condition</strong>, choose <span style="font-family: courier;">Greater&gt;threshold</span>.</li> 
   <li>For <strong>Define the threshold value</strong>, specify the threshold number of failed login attempts that will cause a notification to be sent.<br /> 
    <blockquote>
     <p><strong>Note:</strong> In my example, I’ve specified to be notified after five failed login attempts. You should determine the appropriate threshold to use, based on your organization’s security policies.</p>
    </blockquote> </li> 
  </ol> <p>&nbsp;<br /> </p>
  <div class="wp-caption aligncenter" id="attachment_20979" style="width: 633px;">
   <img alt="Figure 6: Create a CloudWatch alarm" class="size-full wp-image-20979" height="852" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2021/06/26/Monitor-track-failed-logins-6.png" width="623" />
   <p class="wp-caption-text" id="caption-attachment-20979">Figure 6: Create a CloudWatch alarm</p>
  </div> </li> 
 <li>On the <strong>Configure actions </strong>page, choose <strong>Next</strong>.</li> 
 <li>Choose <strong>In alarm</strong>, choose <strong>Select an existing SNS topic</strong>, and then select the SNS topic you created earlier in this procedure.</li> 
 <li>Specify a name for the alarm, and then choose <strong>Create Alarm</strong>.</li> 
</ol> 
<h3>Step 4: Create a Lambda function and subscribe the function to the SNS topic</h3> 
<p>CloudWatch alarm messages are predefined, can’t be modified, and don’t provide details based on CloudWatch streams. Additionally, a CloudWatch alarm will trigger when a combination of failed login attempts on two or more instances meets the threshold. For instance, in my example, when there are three failed attempts on one instance and two failed attempts on a second instance all within a 5-minute period, a CloudWatch alarm will be triggered.</p> 
<p>The purpose of the Lambda function that you’ll create in this step is to validate whether the triggered alarms meet the specified threshold on a per-instance basis before the function sends an email notification to the designated email address. When a CloudWatch alarm is triggered, the function reads through the CloudWatch logs and filters the logs based on CloudWatch log streams that meet the specified threshold for the alarm. If no individual CloudWatch log stream (that is, no individual instance or server) meets the threshold, the function won’t send a notification. The function only sends a notification if it determines that one or more instances have each met the specified threshold. The function also provides more information about the failed login attempts when it does send you an email.</p> 
<h4>To create the Lambda function and subscribe it to the SNS topic</h4> 
<ol> 
 <li>Open the <a href="http://console.aws.amazon.com/lambda" rel="noopener noreferrer" target="_blank">AWS Lambda</a> console and choose <strong>Create function</strong>.</li> 
 <li>Choose <strong>Author from scratch</strong>, and provide a name for your function. Under <strong>Runtime</strong>, select <span style="font-family: courier;">Node.js 14.x</span>, and then choose <strong>Create function</strong>.</li> 
 <li>Double-click <span style="font-family: courier;">index.js</span>, replace the code with the following code, and then choose <strong>Deploy</strong>. 
  <div class="hide-language"> 
   <pre><code class="lang-text">var aws = require('aws-sdk');
var cwl = new aws.CloudWatchLogs();
var ses = new aws.SES();
let alarmThreshold = process.env.ALARM_THRESHOLD;

exports.handler = function(event, context) {
    var message = JSON.parse(event.Records[0].Sns.Message);
    var alarmName = message.AlarmName;
    var oldState = message.OldStateValue;
    var newState = message.NewStateValue;
    var reason = message.NewStateReason;
    var requestParams = {
        metricName: message.Trigger.MetricName,
        metricNamespace: message.Trigger.Namespace
    };
    cwl.describeMetricFilters(requestParams, function(err, data) {
        if(err) console.error('Error is:', err);
        else {
            console.log('Metric Filter data is:', data);
    	    getInstanceIdsAndSendEmail(message, data);
        }
    });
};

function getInstanceIdsAndSendEmail(message, metricFilterData) {
    var timestamp = Date.parse(message.StateChangeTime);
    var offset = message.Trigger.Period * message.Trigger.EvaluationPeriods * 1000;
    var metricFilter = metricFilterData.metricFilters[0];
    var dictInstances = {};
    var arrayInstances = [];
    var instancesFinalList = [];
    var key;
    var val;
    // Getting the Instance Ids
    var paramsForInstanceId = {
        'logGroupName' : metricFilter.logGroupName,
        'filterPattern' : metricFilter.filterPattern ? metricFilter.filterPattern : "",
         'startTime' : timestamp - offset,
         'endTime' : timestamp
    };
    cwl.filterLogEvents(paramsForInstanceId, function (err, data){
        if (err) {
            console.error('Filtering failure:', err);
        } else {
            var events = data.events;
            for (var i in events) {
                var InstanceId = JSON.stringify(events[i]['logStreamName']);
                arrayInstances.push(InstanceId);
            }
            console.log('Array Instance is:', arrayInstances);
            for (var i = 0; i &lt; arrayInstances.length; i++) {
                var instId = arrayInstances[i];
                dictInstances[instId] = dictInstances[instId] ? dictInstances[instId] + 1 : 1;
            }
            console.log('Instance(s) and number of audit failure occurrences:', dictInstances);
            for([key, val] of Object.entries(dictInstances)) {
                if (val &gt; alarmThreshold) {
                    instancesFinalList.push(key.replace(/['"]+/g, ''));
                }
            }
            console.log('Instance(s) with failure audit that exceed the threshold:', instancesFinalList);
    	    getLogsAndSendEmail(message, metricFilterData, instancesFinalList);
        }
    });
}

function getLogsAndSendEmail(message, metricFilterData, logStreamNames_Instance) {
    var timestamp = Date.parse(message.StateChangeTime);
    var offset = message.Trigger.Period * message.Trigger.EvaluationPeriods * 1000;
    var metricFilter = metricFilterData.metricFilters[0];
    var dictInstances = {};
    var arrayInstances = [];
    var instancesFinalList = []

    // Send Email to the Instances
    var paramsForEmail = {
        'logGroupName' : metricFilter.logGroupName,
        'filterPattern' : metricFilter.filterPattern ? metricFilter.filterPattern : "",
         'startTime' : timestamp - offset,
         'endTime' : timestamp,
         'logStreamNames' : logStreamNames_Instance
    };
    cwl.filterLogEvents(paramsForEmail, function (err, data){
        if (err) {
            console.error('Filtering failure:', err);
        } else {
            console.log("===SENDING EMAIL===");
			var email = ses.sendEmail(generateEmailContent(data, message), function(err, data){
                if(err) console.error(err);
                else {
                    console.log("===EMAIL SENT===");
                    console.log(data);
                }
            });
        }
    });
}

function generateEmailContent(data, message) {
    var events = data.events;
    let senderEmail = process.env.SENDER_EMAIL;
    let recipientEmail = process.env.RECIPIENT_EMAIL.split(",");
    console.log('Recipient is: ', recipientEmail);
    console.log('Events are:', events);
    var style = '&lt;style&gt; pre {color: red;} &lt;/style&gt;';
    var logData = '&lt;br/&gt;Logs:&lt;br/&gt;' + style;
    for (var i in events) {
        logData += '&lt;pre&gt;Instance:' + JSON.stringify(events[i]['logStreamName'])  + '&lt;/pre&gt;';
        logData += '&lt;pre&gt;Message:' + JSON.stringify(events[i]['message']) + '&lt;/pre&gt;&lt;br/&gt;';
    }
    var date = new Date(message.StateChangeTime);
    var text = 'Alarm Name: ' + '&lt;b&gt;' + message.AlarmName + '&lt;/b&gt;&lt;br/&gt;' + 
               'Message: ' + 'There has been an unusually high number of Windows Security Audit Failure events for the instance(s) with details below. Please review the event logs &lt;br/&gt;' +
               'Account ID: ' + message.AWSAccountId + '&lt;br/&gt;'+
               'Region: ' + message.Region + '&lt;br/&gt;'+
               'Alarm Time: ' + date.toString() + '&lt;br/&gt;'+
               logData;
    var subject = 'Alarm Triggered - ' + message.AlarmName;
    var emailContent = {
        Destination: {
            ToAddresses: recipientEmail
        },

        Message: {
            Body: {
                Html: {
                    Data: text
                }
            },
            Subject: {
                Data: subject
            }
        },
        Source: senderEmail
    };
    return emailContent;
}
</code></pre> 
  </div> </li> 
 <li>Choose <strong>Add trigger</strong>, and in the drop-down list, choose<strong> SNS</strong>.</li> 
 <li>Under <strong>SNS topic</strong>, select the SNS topic you created in Step 3, and then choose <strong>Add</strong>.<br /> &nbsp;<br /> 
  <div class="wp-caption aligncenter" id="attachment_20980" style="width: 965px;">
   <img alt="Figure 7: Create the AWS Lambda function" class="size-full wp-image-20980" height="599" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2021/06/26/Monitor-track-failed-logins-7.png" width="955" />
   <p class="wp-caption-text" id="caption-attachment-20980">Figure 7: Create the AWS Lambda function</p>
  </div> </li> 
 <li>Choose the<strong> Configuration </strong>tab, and then choose <strong>Environment variables</strong>. Choose<strong> Edit </strong>to add the environment variables for <span style="font-family: courier;">ALARM_THRESHOLD</span>, <span style="font-family: courier;">RECIPIENT_EMAIL</span>, and <span style="font-family: courier;">SENDER_EMAIL</span>, and then choose <strong>Save</strong>.<br /> &nbsp;<br /> 
  <div class="wp-caption aligncenter" id="attachment_20981" style="width: 785px;">
   <img alt="Figure 8: The Lambda environment variables" class="size-full wp-image-20981" height="254" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2021/06/26/Monitor-track-failed-logins-8.png" width="775" />
   <p class="wp-caption-text" id="caption-attachment-20981">Figure 8: The Lambda environment variables</p>
  </div> </li> 
</ol> 
<blockquote>
 <p><strong>Note</strong>: The variables’ keys must be set exactly as <span style="font-family: courier;">ALARM_THRESHOLD</span>, <span style="font-family: courier;">RECIPIENT_EMAIL</span>, and <span style="font-family: courier;">SENDER_EMAIL</span>, because otherwise the code will fail. For the recipient, you can specify a single email or multiple email addresses that are separated by commas, as shown in Figure 8, provided that the emails are verified as specified in the <a href="#Prerequisites">Prerequisites section</a>.</p>
</blockquote> 
<p>Next, create an IAM policy, which you’ll attach to a role that will be assumed by the Lambda function. This policy provides permissions to perform the <span style="font-family: courier;">DescribeMetricFilters</span>, <span style="font-family: courier;">FilterLogEvents</span>, and <span style="font-family: courier;">SendEmail </span>API calls that are necessary for the function to work. It also provides permissions to create a log group and log stream in CloudWatch for the Lambda function, so that you can review the logs if the Lambda function fails to run properly.</p> 
<h4>To create the IAM policy</h4> 
<ol> 
 <li>Sign in to the <a href="https://console.aws.amazon.com/iam" rel="noopener noreferrer" target="_blank">IAM console</a>, and in the navigation bar, choose <strong>Policies</strong>.</li> 
 <li>In the content pane, choose<strong> Create policy</strong>, and then choose<strong> JSON</strong>.</li> 
 <li>Replace the content with the following script. Make sure to replace the placeholders with the ARN of the Lambda function, the ARNs for log group creation and the ARN of your SES verified email address to use as sender. 
  <div class="hide-language"> 
   <pre><code class="lang-text">{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": "ses:SendEmail",
            "Resource": "<em><span style="font-family: courier;"><span style="color: #ff0000;">&lt;arn-of-verified-ses-email-sender&gt;</span></span></em>"
        },
        {
            "Effect": "Allow",
            "Action": [
                "logs:DescribeMetricFilters"
            ],
            "Resource": "<em><span style="font-family: courier;"><span style="color: #ff0000;">&lt;arn-for-CloudWatch-log-groups&gt;</span></span></em>"
        },
        {
            "Effect": "Allow",
            "Action": [
                "logs:FilterLogEvents"
            ],
            "Resource": "<em><span style="font-family: courier;"><span style="color: #ff0000;">&lt;arn-of-CloudWatch-log-group-created-in-step-1&gt;</span></span></em>"

        },
        {
            "Effect": "Allow",
            "Action": "logs:CreateLogGroup",
            "Resource": "<em><span style="font-family: courier;"><span style="color: #ff0000;">&lt;arn-for-CloudWatch-log-groups&gt;</span></span></em>"
        },
        {
            "Effect": "Allow",
            "Action": [
                "logs:CreateLogStream",
                "logs:PutLogEvents"
            ],
            "Resource": [
                "<em><span style="font-family: courier;"><span style="color: #ff0000;">&lt;arn-of-lambda-function:*&gt;</span></span></em>"
            ]
        }
    ]
}
</code></pre> 
  </div> <p>Here is how it appears in my example. Note that <span style="font-family: courier;">FilterSendSecurityEvents</span> is the name of my Lambda function and <span style="font-family: courier;">/aws/SecurityAuditLogs</span> is the name my log group created in Step 1.<br /> &nbsp;<br /> </p>
  <div class="wp-caption aligncenter" id="attachment_20982" style="width: 965px;">
   <img alt="Figure 9: Example policy for the IAM role to be attached to the Lambda function" class="size-full wp-image-20982" height="645" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2021/06/26/Monitor-track-failed-logins-9.png" width="955" />
   <p class="wp-caption-text" id="caption-attachment-20982">Figure 9: Example policy for the IAM role to be attached to the Lambda function</p>
  </div> </li> 
 <li>Choose <strong>Review policy</strong>, specify a name and a description for the policy, and then choose <strong>Create policy</strong>.</li> 
</ol> 
<p>Next, create an IAM role and attach this policy.</p> 
<h4>To create the IAM role and attach the policy</h4> 
<ol> 
 <li>In the <a href="http://console.aws.amazon.com/iam" rel="noopener noreferrer" target="_blank">IAM console</a> navigation bar, choose <strong>Roles</strong>, and then choose<strong> Create role</strong>.</li> 
 <li>Under <strong>Choose the service that will use this role</strong>, choose<strong> Lambda</strong>, and then choose<strong> Next: Permissions</strong>.</li> 
 <li>On the next page, select the policy you just created, and then choose <strong>Next: Tags</strong>. Add an optional tag, and then choose<strong> Next: Review</strong>.</li> 
 <li>Specify a name and description for the role, and then choose <strong>Create role</strong>.</li> 
 <li>To attach this role to the Lambda function, go to the <a href="http://console.aws.amazon.com/lambda" rel="noopener noreferrer" target="_blank">AWS Lambda</a> console. Navigate to the Lambda function, and choose <strong>Configurations</strong>.</li> 
 <li>Choose <strong>Permissions</strong>, and then under <strong>Execution role</strong>, choose <strong>Edit</strong>.</li> 
 <li>On the <strong>Edit basic settings</strong> page, under <strong>Existing role</strong>, select the role you just created, and then choose <strong>Save</strong>.</li> 
</ol> 
<p>And that’s it! You will now be notified whenever there are “Audit Failure” events that reach the threshold you set on a per-instance basis for your AWS Managed Microsoft AD domain-joined instances. If you installed and configured the CloudWatch agent on non–domain-joined instances in <a href="#Step-1">Step 1</a>, then you’ll also get notifications for “Audit Failure” events that are generated by failed login attempts that use local accounts.</p> 
<h2>Conclusion</h2> 
<p>In this post, I showed you how you can proactively track and monitor Windows security audit failures across your <a href="https://aws.amazon.com/directoryservice/" rel="noopener noreferrer" target="_blank">AWS Managed Microsoft AD</a> domain-joined <a href="http://aws.amazon.com/ec2" rel="noopener noreferrer" target="_blank">EC2</a> instances. This helps provide greater visibility into Windows login activities for administrators, so that they can take action to maintain the security of their server fleet. This solution can also be extended to potentially trigger an automation workflow or incident response process in the event of unexpected events.</p> 
<p>Although this blog has specifically targeted AWS Managed Microsoft AD domain-joined instances, the procedure here also applies to standalone EC2 instances or on-premises servers that are configured to send logs to <a href="https://aws.amazon.com/cloudwatch/" rel="noopener noreferrer" target="_blank">CloudWatch</a>.</p> 
<p>If you have feedback about this post, submit comments in the <strong>Comments</strong> section below. If you have questions about this post, start a new thread on the <a href="https://forums.aws.amazon.com/forum.jspa?forumID=180" rel="noopener noreferrer" target="_blank">AWS Directory Service forum</a> or <a href="https://console.aws.amazon.com/support/home" rel="noopener noreferrer" target="_blank" title="contact AWS Support">contact AWS Support</a>.</p> 
<p><strong>Want more AWS Security how-to content, news, and feature announcements? Follow us on <a href="https://twitter.com/AWSsecurityinfo" rel="noopener noreferrer" target="_blank" title="Twitter">Twitter</a>.</strong></p> 
<footer> 
 <div class="blog-author-box"> 
  <div class="blog-author-image">
   <img alt="Author" class="aligncenter size-full wp-image-10920" height="160" src="https://d2908q01vomqb2.cloudfront.net/22d200f8670dbdb3e253a90eee5098477c95c23d/2019/06/19/tekena-author-v3.jpg" width="120" />
  </div> 
  <h3 class="lb-h4">Tekena Orugbani</h3> 
  <p>Tekena is a Cloud Support Engineer at the AWS Cape Town office. He has many years of experience working with Windows Systems, virtualization/cloud technologies, and directory services. When he’s not helping customers make the most of their cloud investments, he enjoys hanging out with his family and watching Premier League football (soccer). </p>
 </div> 
</footer>
