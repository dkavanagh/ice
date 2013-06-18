##Intro
Ice provides a birds-eye view of our large and complex cloud landscape from a usage and cost perspective.  Cloud resources are dynamically provisioned by dozens of service teams within the organization and any static snapshot of resource allocation has limited value.  The ability to trend usage patterns on a global scale, yet decompose them down to a region, availability zone, or service team provides incredible flexibility. Ice allows us to quantify our AWS footprint and to make educated decisions regarding reservation purchases and reallocation of resources.

Ice is a Grails project. It consists of three parts: processor, reader and UI. Processor is responsible to process the Amazon detailed billing file into data readable by reader. Reader is responsible to read data generated by processor and render them to UI. UI is responsible to query reader and render interactive graphs and tables in the browser.

##What it does
Ice communicates with AWS Programmatic Billing Access and maintains knowledge of the following key AWS entity categories:
- Accounts
- Regions
- Services (e.g. EC2, S3, EBS)
- Usage types (e.g. EC2 - m1.xlarge)
- Cost and Usage Categories (On-Demand, Reserved, etc.)
The UI allows you to filter directly on the above categories to custom tailor your view.

In addition, Ice supports the definition of Application Groups. These groups are explicitly defined collections of resources in your organization. Such groups allow usage and cost information to be aggregated by individual service teams within your organization, each consisting of multiple services and resources. Ice also provides the ability to email weekly cost reports for each Application Group showing current usage and past trends.

When representing the cost profile for individual resources, Ice will factor the depreciation schedule into your cost contour, if so desired.  The ability to amortize one-time purchases, such as reservations, over time allows teams to better evaluate their month-to-month cost footprint.

##Screenshots
1. Summary page grouped by accounts
![Summary page grouped by accounts](https://github.com/Netflix/ice/blob/master/screenshots/ss_summary.png?raw=true)

2. Detail page with throughput metrics and grouped by products
![Detail page with throughput metrics and grouped by products](https://github.com/Netflix/ice/blob/master/screenshots/ss_detail.png?raw=true)

3. Reservation page grouped by on-demand, un-used, reserved, upfront costs
![Reservation page](https://github.com/Netflix/ice/blob/master/screenshots/ss_reservation_byreservation.png?raw=true)

4. Reservation page with on-demand cost and grouped by instance types
![Reservation page with on-demand cost and grouped by instance types](https://github.com/Netflix/ice/blob/master/screenshots/ss_reservation_bytype.png?raw=true)

5. Breakdown page of Application Groups
![Breakdown page of Application Groups](https://github.com/Netflix/ice/blob/master/screenshots/ss_breakdown_appgroup.png?raw=true)

##Prerequisite:

1. First sign up for Amazon's programmatic billing access [here](http://docs.aws.amazon.com/awsaccountbilling/latest/about/programaccess.html) to receive detailed billing(hourly) reports. Verify you receive monthly billing file in the following format: `<accountid>-aws-billing-detailed-line-items-<year>-<month>.csv.zip`. If you signed up the beta version of detailed billing file with resources and tag, verify you receive monthly billing file in the this format: `<accountid>-aws-billing-detailed-line-items-with-resources-and-tags-<year>-<month>.csv.zip`.
2. Install Tomcat
3. Install Grail 1.3.7 
4. Get all libraries listed in ivy.xml
5. Ice uses [highstock](http://www.highcharts.com/) to generate interactive graphs. Please make sure you acquire the proper license before using it.
  

##Basic setup: 
Using basic setup, you don't need any extra code change and you will use the provided bootstrap.groovy. You will need to construct your own ice.properties file and have it in your classpath. You can use sample.properties file as the template.

1. Find the s3 billing bucket name and billing file prefix and specify them in ice.properties. For example:
  
          ice.billing_s3bucketname=billing.bucket
          ice.billing_s3bucketprefix=ice/billing/
  
  Tip: If Ice is running from a different account of the s3 billing bucket, for example Ice is running in account "test", while the billing files are written to bucket in account "prod", account "test" may not be able to read those billing files because the AWS Billing user wrote those billing files. In this case, you can either use cross-account roles or create a secondary s3 bucket in account "prod" and grant read access to account "test", and then create a billing file poller running in account "prod" to copy billing files to the secondary bucket.

2. Processor configuration

  2.1 Enable processor in ice.properties:
      
          ice.processor=true
  
  2.2 In ice.properties, set up the local directory where the processor can copy the billing file to and store the output files. For example:
      
          ice.processor.localDir=/mnt/ice_processor

  2.3 In ice.properties, set up the working s3 bucket and working s3 bucket file prefix to upload the processed output files which will be read by reader. For example:
      
          ice.work_s3bucketname=work_s3bucketname
          ice.work_s3bucketprefix=work_s3bucketprefix/

  2.4 Set the following system properties at runtime to access the s3 buckets
     
          ice.s3AccessKeyId=<s3AccessKeyId>
          ice.s3SecretKey=<s3SecretKey>

  2.5 In ice.properties, specify the start time in millis for the processor to start processing billing files. For example, if you want to start processing billing files from April 1, 2013:
      
          ice.startmillis=1364774400000

  2.6 Specify account id and account name mappings in ice.properties, For example:
      
          ice.account.account1=123456789011
          ice.account.account2=123456789012
          ice.account.account3=123456789013

3. Reader configuration

  3.1 Enable reader in ice.properties:
  
          ice.reader=true
      
  3.2 In ice.properties, set up the local directory where the reader will copy files to. For example:
      
          ice.processor.localDir=/mnt/ice_reader
    
    Make sure the local directory is different if you run processor and reader on the same instance.

  3.4 Same as 2.4
  
  3.5 Same as 2.5
  
  3.6 Same as 2.6
  
  3.7 Specify your organization name in ice.properties. This will show up in the UI header.
          ice.companyName=Your Company Name

4. Running Ice

  After the processor and reader setup, you can choose to run the processor and reader on the same or different instances. Running on different instances is recommended.

##Advanced options:
Options with * require writing your own code.

1. Basic reservation service

  If you have reserved instances in your accounts, you may want to make use of the reservation view in the UI, where you can browse/analyze your on-demand, unused reserved instance usage&cost of different instance types in different regions, zones and accounts. in Bootstrap.groovy, BasiicReservationService is used. You can specify reservation period and reservation utilization type in ice.properties:
  
          # reservation period, possible values are oneyear, threeyear
          ice.reservationPeriod=threeyear
          # reservation utilization, possible values are LIGHT, HEAVY
          ice.reservationUtilization=HEAVY

2. Reservation capacity poller

  To use BasiicReservationService, you should also run reservation capacity poller, which will poll different accounts for reservation capacities and store the reservation capacities history in a file in s3 bucket. To run reservation capacity poller, following steps below:
  
    2.1 Set ice.reservationCapacityPoller=true in ice.properties
    
    2.2 Set the following system properties to make describeReservedInstances API call:
    
          ice.reservationAccessKeyId
          ice.reservationSecretKey
      
    2.3 If you IAM account to make the describeReservedInstances API call, also set the following system properties:

          ice.reservationRoleResourceName
          ice.reservationRoleSessionName

3. On-demand instance cost alert

  You can set set an on-demand instance cost threshold so that alert emails will be sent through Amazon SES if the threshold is reached within last 24 hours. The alert emails will be sent no more than once a day. The following properties are needed in ice.properties:
  
          # url prefix, e.g. http://ice.netflix.com/. This is used to create the link in alert email
          ice.urlPrefix=
          # from email address, this email must be registered in ses.
          ice.fromEmail=
          # ec2 ondemand hourly cost threshold to send alert email. The alert email will be sent at most once per day.
          ice.ondemandCostAlertThreshold=250
          # ec2 ondemand hourly cost alert emails, separated by ","
          ice.ondemandCostAlertEmails=

4. Sharing reserved instances among accounts (*)

  If reserved instances are shared among accounts, please specify them in ice.properties. For example:
  
          ice.owneraccount.account1=
          ice.owneraccount.account2=account3,account4
          ice.owneraccount.account5=account6
  
  If different accounts have different AZ mappings, you will also need to subclass BasicAccountService and override method getAccountMappedZone.
   
5. Customized reservation service (*)

  Reserved instance prices in BasiicReservationService are copied from Amazon's ec2 price page as of Jun 1, 2013. Your accounts may have different reservation prices (e.g. Amazon may change prices in the future). In this case, you need to write a subclass of BasicReservationService to provide the correct pricing.

6. Resource service (*)

  If you sign up with Amazon's beta detailed billing file with resources tags, you can use the breakdown feature and application group feature. You will need to implement interface ResourceService and have your own bootstrap.groovy create ProcessorConfig and ReaderConfig.

7. Weekly cost email per application group (*)

  If you have resource service enabled, you can use BasicWeeklyCostEmailService to send weekly cost emails. You can use the default BasicS3ApplicationGroupService, or you can have your own ApplicationGroupService implementation.
  
8. Throughput metric service (*)
   
  You may also want to show your organization's throughput metric alongside usage and cost. You can choose to implement interface ThroughputMetricService, or you can simply use the existing BasicThroughputMetricService. Using BasicThroughputMetricService requires the throughput metric data to be stores monthly in files with names like <filePrefix>_2013_04, <filePrefix>_2013_05. Data in files should be delimited by new lines. <filePrefix> is specified when you create BasicThroughputMetricService instance.



