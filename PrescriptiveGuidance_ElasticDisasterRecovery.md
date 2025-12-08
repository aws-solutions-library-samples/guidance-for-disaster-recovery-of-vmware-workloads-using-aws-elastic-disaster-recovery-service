# VMware to AWS Cloud Native Disaster Recovery Solution Guide

## Overview

This solution guidance document is focused on helping VMware customers transition to a cloud native disaster recovery strategy by using Amazon Web Services as a recovery site. By adopting a cloud native approach for disaster recovery Enterprise customers will see benefits realized through:

- Reduced licensing cost
- No upfront hardware and software investments
- Greater agility in standing up a secondary site
- Lower operational cost by offloading key functions to AWS through the shared responsibility model
- Greater resilience provided through the AWS cloud

In addition, by moving your disaster recovery site to AWS, customers can gain additional flexibility when it comes hypervisor licensing and subscriptions.

Throughout the remainder of this solution guidance document, we provide prescriptive steps to setup a disaster recovery site on AWS using the Elastic Disaster Recovery Service. This guide is meant to help VMware administrators easily design and deploy disaster recovery on AWS. More importantly, by adopting Elastic Disaster Recovery which utilizes a pilot-light approach with a light weight staging environment, administrators will not have to continually manage or pay for the disaster recovery infrastructure. In addition, customer will be able to continuously test and validate their recovery procedures without impacting their production environment. Finally, the approach recommended here, provides an easy onboarding path to a cloud native world.

This guide complements publicly available documentation for Elastic Disaster Recovery, available on the AWS documentation library. It also introduces important concepts and terms to help a VMware administrator get familiar with AWS, provides specific guidance for the configuration of Elastic Disaster Recovery for an on-premises to AWS use case, has step-by-step instructions on how to design, deploy, and manage your Elastic Disaster Recovery implementation and provides code that will help you get started.

## What You'll Learn

By following this guide, you will be able to:

- Understand how VMware technology and terms will map to AWS
- Familiarize yourself with the concepts of Elastic Disaster Recovery
- Learn where Elastic Disaster Recovery fits in your overall disaster recovery design
- Deploy Elastic Disaster Recovery to protect applications that run in an on-premises data center
- Recover source servers in an AWS Region using best practices
- Examine your Elastic Disaster Recovery testing process
- Understand how to recover your servers during a disaster
- Understand how to failback your servers after you have alleviated the disaster at your source environment
- If needed: properly clean up and remove servers from Elastic Disaster Recovery

## What is AWS Elastic Disaster Recovery?

AWS Elastic Disaster Recovery minimizes downtime and data loss with fast, reliable recovery of on-premises and cloud-based applications running on Amazon Elastic Compute Cloud (Amazon EC2) and Amazon Elastic Block Store (Amazon EBS) using affordable storage, minimal compute, and point-in-time recovery. Elastic Disaster Recovery continuously replicates source servers to AWS, allowing you to prepare your environment to recover within minutes from unexpected infrastructure or application outages, human error, data corruption, ransomware, or other disruptions.

Elastic Disaster Recovery provides a unified process to test, recover, and fail back any application running on a supported Operating Systems (OS). Elastic Disaster Recovery supports large, heterogeneous environments with mission-critical workloads. Additionally, this service can support recovery point objectives (RPO) of seconds and recovery time objectives (RTO) of minutes, while reducing disaster recovery infrastructure and operational costs.

## Benefits

With Elastic Disaster Recovery, customers gain the following benefits:

- **Cost Savings**: Save costs by removing idle recovery site resources and pay for full disaster recovery site only when needed; converting fixed costs to variable costs.
  - Fixed costs refer to a company's major, long-term expenses, usually made in a single large purchase upfront.
  - Variable costs refer to a company's day-to-day expenses, usually in the form of multiple smaller payments, such as annual subscriptions, or pay-as-you-go models.

- **Fast Recovery**: Recover applications within minutes to their most up-to-date state or from a previous point in time, for user errors, data corruption, bad patches, ransomware, or other malicious attacks.

- **Unified Process**: Use a unified process to test, recover, and fail back a wide range of OS versions, databases, and applications, without specialized skillsets.

- **Easy Testing**: Perform easy-to-conduct disaster recovery readiness drills without impacting users or your production environment.

- **Flexibility**: Gain flexibility by using AWS as a disaster recovery site with the ability to add or remove replicating servers as needed.

## Architecture

![Architecture Diagram](./assets/images/architecture-diagram.png)

## Core Concepts

In this section, we provide high-level overview of the core concepts that are incorporated in Elastic Disaster Recovery. For a comprehensive understanding of Elastic Disaster Recovery, we recommend becoming familiar with core AWS functionality such as AWS Identity and Access Management (IAM), AWS networking, Amazon EC2, and general disaster recovery concepts.

The main goal of disaster recovery is to help your business prepare and recover from unexpected events in an acceptable amount of time. This means you need to determine which applications deliver the core functionality required for your business to be available, and define the appropriate RTO and recovery point objective RPO required for these applications.

### RPO

Defined by the organization, RPO is the maximum acceptable amount of time since the last data recovery point. This determines what is considered an acceptable loss of data between the last recovery point and the interruption of service.

### RTO

Defined by the organization, RTO is the maximum acceptable time between the interruption of service and restoration of service. This determines what is considered an acceptable time window when service is unavailable after a disaster.

### Source server

The source server refers to the server that you want to protect and recover in the event of a disaster. Elastic Disaster Recovery can protect applications hosted on physical infrastructure, VMware vSphere, Microsoft Hyper-V, and cloud infrastructure from other cloud providers. You can also use Elastic Disaster Recovery to recover Amazon EC2 instances (referred to as Recovery Instances on DRS) in a different Availability Zone or a different AWS Region.

### Recovery subnet

The recovery subnet is the virtual network segment in an AWS Region and Availability Zone that hosts the recovered source servers in the event of a disaster.

### AWS Replication Agent

The AWS Replication Agent is a lightweight software package. It must be installed on each source server that you want to protect using Elastic Disaster Recovery. The agent performs two main tasks: 1) initial block-level replication of disks, by copying the state of the disk on the source server and transmitting this data to the staging environment (where this data is persisted on EBS volumes that map to the source disks), and 2) real-time monitoring and replication of all block-level changes once the agent has completed the initial synchronization process.

### Staging area subnet

In the selected AWS account and Region, the subnet selected to host the replication server is referred to as the staging area subnet. Elastic Disaster Recovery uses low-cost compute and storage hosted on the staging area subnet to keep the data in sync with the source environment. Replication resources consist of replication servers, staging volumes, and snapshots.

### Replication server

The replication server is responsible for receiving and storing the replicated data from the source server. The replication server is an EC2 instance, to which staging EBS volumes are attached. The AWS Replication Agent sends data from the source server to the replication server during the initial synchronization process or when blocks change on the source server. Replication servers will take snapshots of the staging EBS volumes attached to them.

### Point in time (PiT) snapshots

These are periodic backups taken by the replication server at specific intervals to capture the state of the source server and its data. The interval is 1) once every 10 minutes for the last hour, 2) once an hour for the last 24 hours, and 3) once a day for the last 7 days (unless a different retention period is configured, 1-365 days). These PiT snapshots are used during recovery or recovery drill to recover the source server to a particular point in time.

### Conversion server

The conversion server is a component which makes all the necessary modifications to allow the target instance to boot natively in AWS. This includes changes to the drivers, network files, and OS license.

### Drills

Drills refer to scheduled or ad-hoc tests performed to validate the effectiveness of your disaster recovery plan. Elastic Disaster Recovery allows you to conduct drills to simulate recovery scenarios without impacting the production environment or replication state.

### Recovery instance

During an actual recovery, a recovery instance is provisioned in the recovery subnet. The recovery instance is an EC2 instance and a fully functional copy of the source server that allows you to recover operations in the target region.

### Drill instance

A drill instance is an instance that has been launched using Elastic Disaster Recovery, for the purpose of a drill or "test." The goal of launching a drill instance is to test and validate your disaster recovery plan before an actual disaster. This instance is meant to be temporary, and not used for a true disaster, but you can change your mind and keep it as a disaster recovery target if you choose.

### Failover

Failover includes additional steps beyond what Elastic Disaster Recovery currently provides during a recovery or recovery drill. Failover is part of your full disaster recovery strategy and is the process of enacting your disaster recovery playbook in the target Region once a disaster has been declared. Once the failover process is complete, you are now running in the recovery site (target Region). After a failover, the recovery resource is functioning as a stand-in replacement for the original. (It is connected to the rest of the application, DNS, console or infrastructure as code.) Elastic Disaster Recovery covers the launch of the EC2 instances and EBS volumes, but not the rest of failover process, such as configuring the DNS or databases to work with the new instances.

### Failback

Failback is the process of returning to normal operations at your primary site. This includes replicating data back to the source environment, bringing the source servers back online, and redirecting user traffic back to these machines. Redirection of traffic and other configuration operations are handled outside of Elastic Disaster Recovery.
## Planning

Elastic Disaster Recovery is only one part of a larger disaster recovery strategy, and being prepared for these unforeseen events requires proper planning and preparation. The recovery plan should be documented and clearly define stakeholders, with roles and responsibilities, along with the steps that should be taken in the event of a real disaster. Consider the following key concepts as part of your planning process.

### Identify the stakeholders

Identify all individuals and stakeholders who should be involved when a disaster occurs. Consider using tools such as a responsibility matrix that provide a method to define who is responsible, accountable, consulted, and informed during a disaster. In many situations, we tend to focus on technical stakeholders, who are involved with responding to the actual disaster, but we should also consider other stakeholders, such as vendors, third-party suppliers, public relations, marketing teams, and even key customers. We recommend keeping a registry of all stakeholders, with their defined responsibilities and contact information. One of the most critical roles when preparing for a disaster is defining the individual(s) who will make the final decision on declaring a disaster and initiating the business continuity or disaster recovery plan.

### Establish communication channels

Once you have identified and documented all relevant stakeholders, you should define the proper communication channels to keep everyone informed. Part of this process should be establishing a chain of command and defining well-understood escalation paths. We recommend using dedicated communication channels and hubs, such as an on-site situation room where everyone will gather to respond to the disaster. Video conferencing and instant messaging can also be used to facilitate virtual meeting rooms. We strongly recommend keeping executive leadership informed throughout the process.

### Maintain up-to-date documentation

Disaster may be hard to predict, but how we respond to these types of events should be predictable. Once it has been determined that you will be activating your disaster recovery response, it is critical to follow tried and tested procedures. In all cases, this should start with up-to-date documentation detailing all steps to be followed.

The documentation should include: information on configuration state (mapped network connections, with functioning devices and their configurations); the entire setup of systems and their usage, including operating system (OS) and configuration, applications versions, storage and databases (with details such as how and where the data is saved, how backups are restored, how the data is verified for accuracy); architecture diagrams; vendor support contacts; and the responsibility matrix. Your documentation should contain everything IT-related that your business relies upon. Keep hard copies of the documentation, as outages may knock your internal systems offline.

### Determine when to activate the disaster recovery plan

It is important to know as soon as possible that your workloads are not delivering the business outcomes that they should be delivering. As soon as you know, you can quickly declare a disaster and recover from an incident. For aggressive recovery objectives, this response time coupled with appropriate information is critical in meeting recovery objectives. If your recovery point objective is one hour, then you need to detect the incident, notify appropriate personnel, engage your escalation processes, evaluate information (if you have any) on expected time to recovery (without executing the disaster recovery plan), declare a disaster, and recover within an hour.

Key performance indicators (KPIs) are quantifiable measurements that help you understand how well you're performing. It is critical to define and track KPIs to determine when your business processes are impaired and determine the cause. You can catch when KPIs fall below expected thresholds and quickly declare a disaster and recover from an unexpected event. For aggressive recovery objectives, the time to detect an event, declare a disaster, and respond with your recovery plan will determine if your recovery objectives can be met.

### Define action response procedure and verification process

After declaring a disaster, the recovery environment should be activated as soon as possible. An action response procedure outlines all of the necessary steps for recovering at the disaster recovery site. Ensure that your action response procedure is documented and provides details on how the necessary services will be started, verified, and controlled. We recommend using automation whenever possible to minimize the impact of human error. Having all services up in the recovery site is not enough to declare success. It is critical to have a verification process that tests that all of the required data is in place, network traffic has been redirected, and all of the required business applications are functioning properly.

### Perform regular disaster recovery drills

Many organizations do not perform disaster recovery drills on a regular basis because their failover procedures are too complex, and they have concerns that failover tests will lead to a disruption of their production environment (and possibly data loss). Despite these concerns, it is important to schedule frequent disaster recovery drills to build confidence in the plan, build comfort within the team, and identify gaps. People will play a large part in any disaster recovery plan, and only by rehearsing the steps and procedures can you ensure that they respond quickly and accurately to a real event. Further, as the state and configuration of systems change over time, the only way to identify unexpected impact is by conducting these drills. In many cases, planned drills can be scoped down to focus on specific parts of the response plan. When using Elastic Disaster Recovery, you can conduct these drills in an isolated manner so that production is not impacted.

### Stay up to date

Many companies maintain a risk register that tracks and quantifies potential risks to the business. They often include an analysis of current threats, previous disasters, and lessons learned. The risk register should have stakeholders that extend outside of the technology and operations teams and include the business, risk, and executive leadership. It is important to be aware of how you handled previous disasters and how you performed during more recent drills. All documentation should be up to date, reflecting the current environment, processes, and procedures.

### Prepare for failback to primary environment

For most organizations, the disaster recovery site is not designed to run long term, and they will look to failback to the primary site once the disaster has been mitigated. Recovering to the primary site will often be require moving data and business services back. In order to complete this process, you will need to plan for downtime or a partial disruption of your business during the failback process. Elastic Disaster Recovery natively supports failback and simplifies the recovery process to your primary site. The service does not assist with redirecting customer traffic, but this can be automated in most cases using [post-launch actions](https://docs.aws.amazon.com/drs/latest/userguide/post-launch-action-settings-activation-default.html).

## Technical Prerequisites

Implementing Elastic Disaster Recovery is a critical step in ensuring business continuity and resilience against unexpected disruptions. To achieve a successful deployment, it is essential to meet specific technical requirements that encompass various aspects of the system. These requirements range from network settings and communication protocols to supported operating systems, AWS Regions, and installation prerequisites.

The following sections provide a detailed overview of the technical requirements necessary for the implementation of Elastic Disaster Recovery. They include guidelines for staging area subnets, network requirements, Amazon S3 bucket access, operational subnets, supported AWS Regions, general installation requirements, and specific considerations for Windows and Linux systems.

* **Administrative rights** - Elastic Disaster Recovery can only be initialized by the admin user of your AWS Account.
* **Multi-account requirements** (For more details, review [Using multiple staging accounts with AWS DRS](https://docs.aws.amazon.com/drs/latest/userguide/multi-account.html).)
* **Staging account planning and limitations**: Due to AWS account wide API limitations, Elastic Disaster Recovery is limited to protecting 300 source servers per account. In order to replicate more than 300 servers, you would be required to create multiple staging area accounts. It would still be possible to recover all of your servers into a single recovery environment. Elastic Disaster Recovery can recover up to 3,000 servers into a single target account.
* **Network requirements** (For more details, review [Network requirements](https://docs.aws.amazon.com/drs/latest/userguide/Network-Requirements.html).)
    * **Preparation**: Create a dedicated staging subnet for data replication from source servers to AWS. This subnet should have a Classless Inter Domain Routing (CIDR) range that meets the following criteria:
        * Does not overlap with the source server CIDR ranges
        * Has enough IP addresses for replication servers (1 replication server per 15 source volumes, unless dedicated replication servers are need for highly transactional source servers)
        * Has enough IP addresses to support 1 conversion server per launched source server
    * **Staging subnet access requirements**: The staging area subnet requires outbound access to the Amazon EC2, Amazon Simple Storage Service (Amazon S3), and Elastic Disaster Recovery endpoints, within the target Region.
    * **Communication over TCP Port 443**: All communication is encrypted with TLS. All control plane traffic is handled over TCP port 443 and should be permitted for the following:
        * Between the source servers and Elastic Disaster Recovery Service
        * Between the staging area subnet and AWS Elastic Disaster Recovery
        * The Elastic Disaster Recovery AWS Region-specific Console address:
            * (drs.[region].amazonaws.com example: drs.eu-west-1.amazonaws.com)
        * Amazon S3 service URLs (required for downloading Elastic Disaster Recovery software)
        * The AWS Replication Agent installer should have access to the S3 bucket URL of the AWS Region you are using with Elastic Disaster Recovery.
        * The staging area subnet should have access to the regional S3 endpoint.
        * The staging area subnet requires outbound access to the Amazon EC2 endpoint of its AWS Region.
    * **Communication over TCP Port 1500**: All data replication traffic is transmitted between the source servers and the staging area subnet using TCP Port 1500. This communication is also encrypted.
    * **Bandwidth Requirements**: Ensure that the average network bandwidth is higher than the write rate on the source servers for successful replication.
* **S3 buckets** (For more details, review [Network requirements](https://docs.aws.amazon.com/drs/latest/userguide/Network-Requirements.html).)
    * **Access requirements**: Agent installation and replication server components require S3 bucket access.
    * **VPC endpoint policy**: Ensure that the relevant VPC endpoint policy includes access to all required S3 buckets. Refer to the example policy for replicating to us-east-1 and Amazon S3 documentation for policy requirements.
* **Operational subnets** (For more details, review [Network requirements](https://docs.aws.amazon.com/drs/latest/userguide/Network-Requirements.html).)
    * **Drill and recovery subnets**: You should create recovery subnets (and optionally drill subnets) before attempting to launch recovery instances. Instances are launched in a subnet specified in the Amazon EC2 launch template associated with each source server.
* **Supported Elastic Disaster Recovery AWS Regions**
    * Review [Elastic Disaster Recovery supported AWS Regions](https://docs.aws.amazon.com/drs/latest/userguide/supported-regions.html) for an up-to-date list of all supported Regions.
* **Supported operating systems**
    * Elastic Disaster Recovery supports many versions of Windows and Linux operating systems, some of which are not natively supported by Amazon EC2. Review an up-to-date version of [supported operating systems](https://docs.aws.amazon.com/drs/latest/userguide/adding-servers.html).
* **Windows installation requirements** (For more details, review [Windows installation requirements](https://docs.aws.amazon.com/drs/latest/userguide/windows-agent.html).)
    * **Supported operating systems**: Ensure that your source server operating system is supported.
    * **Free disk space**: Confirm that you have at least 4 GB of free disk space on the root directory (C:\Windows by default).
    * **Free RAM**: Confirm that you have at least 300 MB of free RAM.
    * **MAC address stability**: Ensure that the MAC addresses of the source servers do not change upon a reboot or any other common changes in your network environment. The Elastic Disaster Recovery agent may use the MAC address in its process to link the source server to its replication infrastructure.
* **Linux installation requirements** (For more details, review [Linux installation requirements](https://docs.aws.amazon.com/drs/latest/userguide/linux-agent.html).)
    * **Supported Operating Systems**: Ensure that your source server operating system is supported (referenced above)
    * **MAC Address stability**: Ensure that the MAC addresses of the source servers do not change upon a reboot or any other common changes in your network environment. The Elastic Disaster Recovery agent may use the MAC address in its process to link the source server to its replication infrastructure.
    * **Python**: Python 2 (2.4 or above) or Python 3 (3.0 or above) must be installed on the server.
    * **Free Disk Space**: At least 4 GB on the root directory (/), 500 MB on the /tmp directory.
    * **GRUB Bootloader**: The active bootloader software must be GRUB 1 or 2.
    * **/tmp Directory**: Mounted as read+write and with the exec option.
    * **Sudoers List**: The Linux account that is installing Elastic Disaster Recovery needs to be in the sudoers list.
    * **dhclient Package**: Ensure that the dhclient package is installed.
    * **Kernel Headers**: Verify that kernel-devel/linux-headers are installed and match the running kernel version.
    * **Symbolic Link Considerations**: Ensure that the content of the kernel-devel/linux-headers is not a symbolic link.
        * Sometimes, the content of the kernel-devel/linux-headers, which match the version of the kernel, is actually a symbolic link. In this case, you will need to remove the link before installing the required package. To verify that the folder that contains the kernel-devel/linux-headers is not a symbolic link, run the following command:
            * On RHEL/CENTOS/Oracle: ls -l /usr/src/kernels
            * On Debian/Ubuntu/SUSE: ls -l /usr/src
        * If you found that the content of the kernel-devel/linux-headers, which matches the version of the kernel, is a symbolic link, you need to delete the link. Run the following command:
            * rm /usr/src/[LINK NAME]
                * For example: rm /usr/src/linux-headers-4.4.1
        * **Kernel headers installation**: For the agent to operate properly, you need to install a kernel headers package with the exact same version number of the running kernel.
            * To install the correct kernel-devel/linux-headers, run the following command:
                * On RHEL/CENTOS/Oracle/SUSE: sudo yum install kernel-devel-`uname -r`
                * On Debian/Ubuntu: sudo apt-get install linux-headers-`uname -r`
        * If no matching package was found on the repositories configured on your server, you can download it manually from the Internet and then install it. To download the matching kernel-devel/linux-headers package, navigate to the following sites:
            * [RHEL, CENTOS, Oracle, and SUSE package directory](https://rpm.pbone.net/)
            * [Debian package directory](https://www.debian.org/distrib/packages)
            * [Ubuntu package directory](https://packages.ubuntu.com/)
* **AWS-specific considerations**
    * **Number of disks per server**
        * Elastic Disaster Recovery uses Amazon EBS and Amazon EC2 for the replication infrastructure. Because of this, Elastic Disaster Recovery is limited to the amount of disks that can be added to the replication servers: * For Nitro replication instances (such as t3.small), we are limited to source servers with less than 26 volumes. * For Xen replication instances (such as t2.small), the limitation is 40 volumes per source server.
    * **Maximum source disk size**
        * Elastic Disaster Recovery uses Amazon EBS and Amazon EC2 for the replication infrastructure. Because of this, Elastic Disaster Recovery is limited to the 16TB for each disk on the source machines being protected.

## Design Guidance

Consider the following concepts when designing your implementation.

* **Security**
    * Security needs to be a high priority, especially when it comes to your disaster recovery solution. Elastic Disaster Recovery has several security options built directly into the service, however, it does not provide a full security solution. As such, you should work with your security teams to validate your security posture.
    * **Encryption in transit**: All data replicated by Elastic Disaster Recovery is encrypted in transit using TLS 1.2.
    * **Encryption at rest**: All EBS volumes that Elastic Disaster Recovery creates in the staging area are automatically encrypted by default with an [AWS Key Management Service (AWS KMS)](https://docs.aws.amazon.com/kms/latest/developerguide/overview.html) key that the service creates in your account. You can also choose an existing customer managed key (CMK) or create one for this purpose if needed. You must select your preferred key in the EBS encryption section of the replication settings for Elastic Disaster Recovery to use it. EBS volumes that are launched during a drill or recovery will be encrypted using the same key, unless otherwise specified in the EC2 Launch Template.
        * If you have specific compliance requirements, you can also use CMKs instead of the default keys created by Elastic Disaster Recovery to handle the encryption of the staging volumes in addition to the volumes of drill or recovery instances.
    * **Separate DR account**: As a best practice for Elastic Disaster Recovery, use separate AWS accounts for the Elastic Disaster Recovery staging network (virtual private cloud [VPC] and subnet) and recovery network. Using a separate AWS account specifically for your disaster recovery plan allows for better segmentation and separation of your critical replicated data.

* **Networking**
    * Address both connectivity and bandwidth when planning your networking resources:
    * **Network connectivity**: We recommend your prioritize planning the path through which your data will be replicated before starting the installation of the Elastic Disaster Recovery agent. The service can replicate over public or private internet connections using public internet, VPN, [AWS Direct Connect](https://docs.aws.amazon.com/directconnect/latest/userguide/Welcome.html), and [AWS Transit Gateway](https://docs.aws.amazon.com/vpc/latest/tgw/what-is-transit-gateway.html). Understanding which replication method you would like to use ahead of time will prevent you from having to adjust your replication settings in the future. As the number of source servers you are protecting grows, consider using AWS Direct Connect to maximize the stability of the replication over public or virtual private network (VPN)-based replication.
    * **Network bandwidth**: Elastic Disaster Recovery will use as much of the network as possible when replicating data from your source environment. Because of this, you will want to ensure you have enough bandwidth to support your source change rate (ensuring you can maintain continuous data protection), and you will want to monitor your network to ensure there is no congestion caused by the replication process. If you need to throttle the Elastic Disaster Recovery service, you can do so at the service or machine level. To calculate the bandwidth required for your particular workloads, review [Calculating the required bandwidth for TCP Port 1500](https://docs.aws.amazon.com/drs/latest/userguide/Network-Requirements.html#bandwidth-requirements).

## Installation

For a quick overview of the steps, please see [this link](https://docs.aws.amazon.com/drs/latest/userguide/quick-start-guide-gs.html).

1. The first step of installation is to initialize the Elastic Disaster Recovery service in the region that you are planning to recover in.

    a. Elastic Disaster Recovery can only be initialized by the Admin user of your AWS Account.
    
    b. During initialization, the following IAM roles will be created:
    
     * AWSServiceRoleForElasticDisasterRecovery
        * This role allows the Elastic Disaster Recovery service to function properly
        
    * AWSElasticDisasterRecoveryReplicationServerRole
        * This creates the role attached to Replication Servers
        
    * AWSElasticDisasterRecoveryConversionServerRole
        * This creates the role for Conversion Servers
        
    * AWSElasticDisasterRecoveryRecoveryInstanceRole
        * This is the role that is attached to Recovery Instances when they are launched
        
    * AWSElasticDisasterRecoveryAgentRole
        * This role is used to manage the operations required by the AWS replication agent to function properly
        
    * AWSElasticDisasterRecoveryFailbackRole
        * This role is used in the recovery portion of failback

2. For security, we recommend using AWS managed policies to add permissions to users, groups, and roles. Even though AWS provides the ability to write custom policies and [create IAM customer managed policies](https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies_create-console.html), these actions may require additional time and expertise. To get started quickly, use AWS managed policies. These policies cover common use cases and are available in your AWS account. For more information about AWS managed policies, review [AWS managed policies in the IAM User Guide](https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies_managed-vs-inline.html#aws-managed-policies). AWS services maintain and update AWS managed policies. You can't change the permissions in AWS managed policies. Along with the IAM roles above, Elastic Disaster Recovery will also provide access to the following IAM policies:

    a. [AWSElasticDisasterRecoveryConsoleFullAccess](https://docs.aws.amazon.com/drs/latest/userguide/what-is-drs.html)
    
    * If you are using the AWS console, this policy provides you with the required permissions, including full access to all public APIs of Elastic Disaster Recovery and permissions to read the KMS key, License Manager, Resource Groups, Elastic Load Balancing (ELB), IAM, and Amazon EC2 information. It also includes Amazon EC2 actions that allow to launch, delete, or modify replication servers and recovery instances. Attach this policy to your users or roles.
    
    b. [AWSElasticDisasterRecoveryReadOnlyAccess](https://docs.aws.amazon.com/drs/latest/userguide/security-iam-awsmanpol-AWSElasticDisasterRecoveryReadOnlyAccess.html)
    
    * You can attach the AWSElasticDisasterRecoveryReadOnlyAccess policy to your IAM identities.
    * If you are using the AWS console, this policy provides you with the required read-only permissions to all read-only public APIs of Elastic Disaster Recovery in addition to some read-only APIs of other AWS services that are required in order to make full read-only use of the Elastic Disaster Recovery console.
    
    c. [AWSElasticDisasterRecoveryAgentPolicy](https://docs.aws.amazon.com/drs/latest/userguide/security-iam-awsmanpol-AWSElasticDisasterRecoveryAgentPolicy.html)
    * This policy permits the use of AWS Replication Agent, which is used with Elastic Disaster Recovery to recover source servers to AWS. We do not recommend that you attach this policy to your users or roles.
    
    d. [AWSElasticDisasterRecoveryAgentInstallationPolicy](https://docs.aws.amazon.com/drs/latest/userguide/security-iam-awsmanpol-AWSElasticDisasterRecoveryAgentInstallationPolicy.html)
    
    * This policy allows installation of the AWS Replication Agent, which is used with Elastic Disaster Recovery to recover external servers to AWS. Attach this policy to your IAM users or roles whose credentials you provide during the installation step of the AWS Replication Agent.
    
    e. [AWSElasticDisasterRecoveryFailbackPolicy](https://docs.aws.amazon.com/drs/latest/userguide/security-iam-awsmanpol-AWSElasticDisasterRecoveryFailbackPolicy.html)
    
    * This policy allows setup of the Elastic Disaster Recovery Failback Client, which is used to failback recovery instances back to your original source infrastructure. We do not recommend that you attach this policy to your users or roles.
    * This policy is used by Elastic Disaster Recovery to refresh credentials for the Elastic Disaster Recovery Failback Client. We do not recommend that you attach this policy to your users or roles.
    
    f. [AWSElasticDisasterRecoveryFailbackInstallationPolicy](https://docs.aws.amazon.com/drs/latest/userguide/security-iam-awsmanpol-AWSElasticDisasterRecoveryFailbackInstallationPolicy.html)
    
    * You can attach the AWSElasticDisasterRecoveryFailbackInstallationPolicy policy to your IAM identities.
    * This policy allows installation of the Elastic Disaster Recovery Failback Client, which is used to failback recovery instances back to your original source infrastructure. Attach this policy to your IAM users or roles whose credentials you provide when running the Elastic Disaster Recovery Failback Client.
    
    g. [AWSElasticDisasterRecoveryRecoveryInstancePolicy](https://docs.aws.amazon.com/drs/latest/userguide/security-iam-awsmanpol-AWSElasticDisasterRecoveryRecoveryInstancePolicy.html)
    
    * This policy is attached to the instance role of Elastic Disaster Recovery's recovery instance.
    * This policy allows the Elastic Disaster Recovery recovery instance, which are EC2 instances launched by Elastic Disaster Recovery, to communicate with the Elastic Disaster Recovery service and to be able to failback to their original source infrastructure. An IAM role with this policy is attached (as an EC2 Instance Profile) by Elastic Disaster Recovery to the recovery instances. We do not recommend that you attach this policy to your users or roles.
    
    h. [AWSElasticDisasterRecoveryServiceRolePolicy](https://docs.aws.amazon.com/drs/latest/userguide/security-iam-awsmanpol-AWSElasticDisasterRecoveryServiceRolePolicy.html)
    
    * This policy allows Elastic Disaster Recovery to manage AWS resources on your behalf.
    * This policy is attached to the AWSServiceRoleForElasticDisasterRecovery role.
        * This policy includes permissions to do the following:   
            * **ec2** – Retrieve and modify resources needed to support failover and failback of source servers and source networks
            * **cloudwatch** – Retrieve disk usage to allow cost optimization
            * **iam** – Acquire the permissions required for recovery
            * **kms** – Allow using encrypted volumes
            * **drs** – Retrieve tags and set tags for Elastic Disaster Recovery resources, create Elastic Disaster Recovery resources on failover
    
    i. For a full list of AWS of policies Elastic Disaster Recovery can create, or to see the please see [AWS managed policies for Elastic Disaster Recovery](https://docs.aws.amazon.com/drs/latest/userguide/security-iam-awsmanpol.html)

3. Next, you need to create the IAM role that will be used to install the Elastic Disaster Recovery agent. We recommend temporary credentials that can be used for installation and discarded after the agent has been installed.

    a. **Note**: Using AWS Security Token Service (AWS STS), you can create or provide temporary security credentials that control access to your AWS resources. Temporary security credentials work almost identically to a user's long-term access key credentials for IAM, with the exception of two differences: First, temporary credentials are short term, so they expire after a time that you configure. After they expire, AWS no longer recognizes them or allows any kind of access from API requests made with them. Second, they are not stored, but are generated dynamically and provided when requested.

    b. **Note**: When your source server is not on AWS (such as a physical or virtual server on-premises), then you will need to create an IAM role with the necessary permissions and then assume that role from the on-premises server. We recommend using AWS Single Sign-On (AWS SSO), a service that enables you to quickly and easily assign and manage your on-premises employees' access to multiple AWS accounts for identity management. AWS SSO provides the option to create users directly, or you can bring your users from existing identity sources like Microsoft Active Directory, Okta Universal Directory, or any another supported identity provider (IdP).

4. Install the AWS Replication Agent using AWS SSO. On AWS, enable AWS SSO and create users.
    
5. On the source server, configure AWS SSO and log in.
    
6. On the source server, assume the role, [generate temporary credentials](https://docs.aws.amazon.com/mgn/latest/ug/credentials.html) using AWS STS, and [install the agent](https://docs.aws.amazon.com/mgn/latest/ug/agent-installation.html).
    
    a. Elastic Disaster Recovery is installed on a per-server basis. We recommend installation of the agent using temporary credentials. If using AWS Systems Manager, this can be done through an AWS Systems Manager (SSM) document that AWS provides. If you are not using Systems Manager, you can complete the installation manually. Review the below links for step-by-step instruction for either Windows or Linux.
       
    b. Review [Manual installation instructions](https://docs.aws.amazon.com/drs/latest/userguide/adding-servers.html).
       
    * **Note**: If you are using a third-party software deployment process, first consult with the team that manages the software to find out if it can be used to deploy Elastic Disaster Recovery.
       
    c. When installing the Elastic Disaster Recovery agent, you may run into unforeseen installation issues based on multiple factors. Review the troubleshooting guide in the Resources section for information on common issues that can be encountered during the installation process. If you are unable to resolve the using the information provided, create an AWS support ticket and include the following information:
       
    * What part of the installation process is failing
    * Confirmation that you have followed the troubleshooting guide
    * Attach the agent log from that specific server
    * The agent log can be located at the following locations: [Where can I find the AWS DRS Replication Agent logs](https://docs.aws.amazon.com/drs/latest/userguide/Agent-Related-FAQ.html#agent-logs-location)

7. Once the installation is complete, the Elastic Disaster Recovery console will show the following stages:

    a. Initiating: This shows that the agent has been installed successfully on the source server, and Elastic Disaster Recovery is now moving on to the next steps of configuring replication for that server. To see what step the service is currently on, select the server name, and check under Data replication status, as shown here.
    
    b. Initial sync, {xx% done} time left: This is the amount of the known blocks that will be replicated.
    
    * You may see the time left for replication fluctuate by large margins. This is due to the way that reading block storage is accomplished, and we are unable to predict how many future blocks may need to be replicated.
    * You can estimate the amount of time required to complete this step by analyzing the amount of storage that needs to be replicated and the available bandwidth available to transmit this data.
    * During this initial sync process, you may see backlog in the same line. Backlog is the amount of new data that has been written, waiting to be added after initial sync. Once initial sync has completed, you will see the backlog amount start to reduce as the agent replicates those newer blocks.
    
    c. Initial sync 100% done: Create a snapshot.
    
    * All blocks have been replicated from the source machine to the staging area, and you will now create the baseline EBS snapshot for that volume.
    * If the service is stuck in the stage for a long time, confirm the replication server has 443 outbound access to a regional EC2 endpoint.
    
    d. Healthy: All data has been replicated to the staging area, and the replication server has enough bandwidth to replicate the changes being generated at the source environment
    
    e. Should there be an issue with the replication process after you have completed the initial sync phase, you will see an error in the same location.
    
    f. Other states you might see are:
    
    * Rescan: This means that something has interrupted the agents ability to validate the block map, usually caused by an unplanned reboot of the source machine (such as a power outage, pulling the plug, or terminating an EC2 instance)
    * Lag: Lag is the amount of time since the server was last in continuous data protection (CDP) mode This can be caused by many items, and troubleshooting steps can be found in [Other troubleshooting topics](https://docs.aws.amazon.com/drs/latest/userguide/what-is-drs.html).
    * Backlog: Backlog is the amount of data that was written to the disk and still needs to be replicated in order to reach CDP mode. Backlog can be caused by the initial sync process (this will clear up on its own), a burst of large writes, or other unknown factors. The agent will continue to work through the backlog automatically.

8. Once the installation process has been completed across all needed servers, you can move on to the next section, where you will configure monitoring and notifications.

## Monitoring

Monitoring will play a critical role when defining a disaster recovery strategy. It is crucial to observe, monitor, and alert on resources and system performance at multiple levels to be able to move forward with your disaster recovery operations.

### Configure replication monitoring and alerting

Elastic Disaster Recovery can use Amazon CloudWatch to assist with monitoring of your disaster recovery solution. CloudWatch is a monitoring service that helps you monitor AWS resources as they are consumed within your account. When these two services are integrated, you can monitor Elastic Disaster Recovery events with Amazon CloudWatch to build a customizable and detailed dashboard for Elastic Disaster Recovery. These services can be extended further with Amazon EventBridge and Amazon Simple Notification Service (Amazon SNS), to get real-time alerts and to automate responses.

### Creating CloudWatch dashboards to monitor Elastic Disaster Recovery

You can visualize and share your metrics using CloudWatch dashboards. There are many metrics available within CloudWatch to help you monitor and manage the state of your disaster recovery operations. With CloudWatch, you can include metrics to monitor your source server count, time since last successful test, and lag of source servers (when Elastic Disaster Recovery is no longer in continuous data protection mode and should be investigated for root cause). We recommend that you use CloudWatch to set up dashboards and notifications to alert you on any possible replication issues as detailed in the steps below:

1. Navigate to Amazon CloudWatch dashboard.

2. Under Dashboards, select Automatic dashboards.

3. Filter for and select Elastic Recovery Service.

    a. You will be taken to a default dashboard that monitors several aspects of Elastic Disaster Recovery. These metrics are based on the replication instances you have running in the AWS Region you currently have selected. The metrics are listed below:
    
    * LagDuration: Average: This is the average time of "Lag" on your replication severs. Anything higher than 0 should be investigated for possible issues, but we recommend monitoring for lags larger than an hour (or your RPO if close to an hour).
        
    * Backlog: Average: This is the average amount of "backlog". Backlog is generated when the service is unhealthy but is still seeing data being written to source that it is unable to replicate.
        
    * DurationSinceLastSuccessfulRecoveryLaunch: Maximum: This is the maximum amount of time since the last successful launch of Elastic Disaster Recovery machines.
        
    * ElapsedReplicationDuration: Maximum: This is the amount of time Elastic Disaster Recovery has been replicating data.
        
    * ActiveSourceServerCount: Average: This is how many source servers have had Elastic Disaster Recovery installed on them and are currently replicating data
        
    * TotalSourceServerCount: Average: This is how many source servers have had Elastic Disaster Recovery installed on them.

4. Choose Add to dashboard.

    a. You can either select an existing dashboard or create one by choosing Create new.
    
    * If you decide to create a new dashboard, you will be taken to the next screen to enter a name. Enter a name, and select Create.
    
    b. Select Add to dashboard.

5. You will now have a dashboard monitoring Elastic Disaster Recovery under your Custom dashboards section in CloudWatch.

### Configuring your Amazon SNS topic

Amazon SNS will be used to alert a specific inbox or distribution list of any Elastic Disaster Recovery source machines with stalled replication that must be addressed. Doing so will help to identify and remediate issues quicker, so that you can maintain your RPO goals. Stalled replication is the main indicator of replication issues and other issues.

1. Navigate to Amazon Simple Notification Service.

2. Choose Create Topic.

    a. Under Details and Type, choose Standard.
    
    b. Under Name, enter a name for this topic (for this example, we will use "drs-replication-monitoring").
    
    c. Optional: Enter a display name for SMS messages to mobile devices.
    
    * **NOTE**: As of June 1, 2021, US telecom providers no longer support person-to-person long codes for applications-to-person communications. See the Amazon SNS Developer Guide for more information.
    
    d. Optional: For Tags, enter a key-value pair for easy identification later on.

3. Select Create topic.

    a. Once the topic is created, select drs-replication-monitoring from the list.
    
    b. Choose Create subscription.
    
    c. Validate that the Topic ARN under Details is the same as drs-in-lag.
    
    d. From the Protocol dropdown, choose email.
    
    e. Under Endpoint, add the email or distribution list to receive these alerts.
    
    f. Choose Create subscription.

### Create a rule using the console

The next step is to configure EventBridge to monitor for specific Elastic Disaster Recovery events related to replication health. If EventBridge receives an event for unhealthy replication status for Elastic Disaster Recovery, it will notify the Amazon SNS topic. This, in turn, notifies the subscribers of that topic. To create an EventBridge rule to monitor these events, follow these rules:

1. Open Amazon EventBridge.

2. Choose Create rule.

    a. Under Name and description and Name, enter the name for this rule. (we use "drs-replication-monitoring" for this step)
    
    b. Under Define pattern, choose Event pattern.
    
    * Select Pre-defined pattern by service.
        
        * From the dropdown menu for Service provider, choose AWS.
        * Under the Service name dropdown, choose Elastic Disaster Recovery Service.
        * Under Event type, choose DRS Source Server Data Replication Stalled Change.
        
    * Under Select targets and Target, choose SNS topic.
        
        * For Topic, choose the SNS topic created earlier, drs-replication-monitoring.
        
        * Choose Create.

You have now created a dashboard that will monitor your Elastic Disaster Recovery replication infrastructure and will notify you if there are any stalled replication servers that would cause you to miss your RPO.

## Cost Monitoring

The following sections provide considerations for cost monitoring.

### Configure cost monitoring

There are several configuration strategies possible with Elastic Disaster Recovery. Understanding what makes up the associated costs of using Elastic Disaster Recovery is an important consideration when deciding how to further optimize the system for performance versus cost, while still maintaining your resilience objectives. This may include decisions on retention periods, Region selection, network design, and infrastructure configurations.

The following section provides the steps to activate cost allocation tags, to create and save a custom report, and to export the report data. This will provide insight into the overall costs of the Amazon EC2, Amazon EBS, and EBS snapshot resources provisioned by Elastic Disaster Recovery.

### Activate cost allocation tags

This section details the process of enabling user-defined cost allocation tags for Elastic Disaster Recovery. Once enabled, you can use these tags on your cost allocation report to track costs.

1. Log in to the AWS Management Console and search for Billing.

2. Select cost allocation tags on the left.

3. Under User-defined cost allocation tags, find the AWSElasticDisasterRecoveryManaged tag.

4. Select the checkbox for this tag, and choose Activate in the top right.

5. Choose Activate from the pop up. It may take a couple of hours before the tags are available.

6. Optimize in the following ways:

    a. Use Default instance types for replication servers unless source servers are often in lag.
    
    b. Use automated disk type.
    
    c. Lower snapshot retention to minimal needs.
    
    **NOTE**: Elastic Disaster Recovery should not be used for long term retention, and you should use a backup solution for long term storage.

### Create cost categories

This section details the process of creating cost categories. Cost categories allow you to map Elastic Disaster Recovery costs and usage into meaningful categories using a rules-based engine.

1. Log in to the AWS Management Console, and search for Billing.

2. Select Cost Categories on the left, and then select Create Cost Category.

3. Provide a Name (for example, "DRSCost") to the cost category, and select Next.

4. Choose Rule type as Inherited value, and Dimension as Cost Allocation Tag, Tag key as AWSElasticDisasterRecoveryManaged, and select Add rule.

    a. Choose Rule type as Regular and DRSCost Value as License.
    
    b. Under the Dimension 1 section, choose Service. For Operator, select Is and for Service Code, choose AWSElasticDisasterRecovery. Then, select Next.

5. Select Create cost category. It will take up to 24 hours for the cost category to be available in AWS Cost Explorer.

### Create Cost Explorer report

This section details the steps to create a customized Cost Explorer report for Elastic Disaster Recovery. It uses the filters and cost categories created in the preceding section.

1. Log in to the AWS Management Console, and search for AWS Cost Explorer. Open the AWS Cost Management dashboard, and select Cost Explorer.

2. Under FILTERS, select Cost Category.

    a. Select the cost category (for example, "DRSCost") which was created in the Create Cost Categories section.
    
    b. Select two checkboxes: License and drs.amazonaws.com. Select Apply Filters.

3. On the top left, select the Group by: Usage Type.

4. Go to the time ranges, and select the time for which you would like to see the data. In the following example, we set it to Last 7 Days with the time granularity as Daily.

5. Select Bar style type for the chart.

6. You will see the cost breakdown of the staging area. This includes the cost of replication servers, EBS volumes, EBS snapshots, and other services and AWS resources.

7. Select Save as, which is located near the top left corner, and assign the new report a name (for example, "Elastic Disaster Recovery Service Costs_[date]"). Select Save Report.

### View and export saved Cost Explorer report

This section details the steps to view the Cost Explorer report and export it to a CSV file that can be shared with stakeholders.

1. Log in to the AWS Management Console.

2. Search for AWS Cost Explorer, and open the AWS Cost Management dashboard.

3. Select Reports.

    a. Select the report that was previously saved.
    
    b. The total cost of Elastic Disaster Recovery is included in that report.
    
    c. The report can be further customized by using the Group by options near the top or any of the other filters available in AWS Cost Explorer.

4. Export your data for further analysis by choosing the Download CSV button. Download the CSV file to a location on your computer.

### Optimizing costs for Elastic Disaster Recovery

This section details the steps you can take to enhance cost optimization for Elastic Disaster Recovery workloads.

1. Use the default replication server instance types at first. Allow Elastic Disaster Recovery to replicate your initial dataset, then ensure no source servers are stating that they are in Lag. If there are any source servers that are in lag, please follow the troubleshooting section at the end of this implementation guide.

2. Use the Auto volume type selection option for your replication servers.

    a. When choosing Auto volume type selection, the service will dynamically switch between performance/cost optimized volume type according to the replicated disk write throughput.

3. Lower snapshot retention to minimal length requirements. Based on the changed rate of your dataset, this can have a large impact on overall Elastic Disaster Recovery costs.

    a. If you have compliance requirements and require your snapshots for long-term retention, you should use a long-term storage and backup solution like AWS Backup. Elastic Disaster Recovery is not intended to act as a backup or archive storage service, and hence is not a suitable solution for long-term retention of your snapshots and other data.

### Cost optimization

The method to reduce operational costs when using Elastic Disaster Recovery is to perform a combination of the following:

1. Consider servers on a case-by-case basis. Do they require an ongoing replication-based disaster recovery approach to meet their RPO/RTO objectives?

2. Evaluate the retention period required for point-in-time snapshots. How far into the past do you need to retain the ability to restore? Make sure to consider applicable compliance and regulatory requirements.

3. For those servers being covered by Elastic Disaster Recovery, consider whether there are redundant drives mounted that are no longer in use and do not need to be replicated. These can either be unmounted or excluded when setting up the replication agent.

4. Right-size the target failover infrastructure by selecting the appropriate EC2 instance type in the EC2 launch template. Use the instance right-sizing feature to map to an instance type that closely follows the source infrastructure. Alternatively, you can use operational data in the source environment to further trim resources. For example, consider trimming a server that you know is over provisioned and never exceeds 25% CPU utilization.

The size of the underlying disks (such as the entire disk and not just partitions) directly dictates the amount of data that is replicated to AWS during the initial sync process. As a result, right sizing and being selective of workloads, as per RPO/RTO objectives, gives the benefit of both monetary and saving time.

For more information, review the AWS Well-Architected Cost Optimization Pillar and the AWS Well Architected Performance Efficiency Pillar.

## Drill Planning

In some situations, the disaster recovery test will be a failover of the production environment in a planned event with apps running in production in the recovery Region. We advise doing a full production test on an annual basis to capture any blockers and to be familiar with the process in the event of an actual disaster.

### Drills versus planned disaster recovery events

Testing your disaster recovery implementation is the only way to validate that your RPO and RTO objectives can be met when a real disaster occurs. Elastic Disaster Recovery natively supports the ability to launch drills without affecting your production environment. However, conducting a drill and launching a server as an EC2 instance is not adequate to declare success. It's important to test at an application or business process level to ensure that the end to end service can be delivered when the disaster recovery plan is activated. It is a best practice to perform drills regularly. There are a few things to note before launching an Elastic Disaster Recovery drill:

1. When launching a drill or recovery, you can launch up to 500 source servers in a single operation. Additional source servers can be launched in subsequent operations.

2. It is a best practice to perform drills regularly. After launching drill instances, use either SSH (Linux) or RDP (Windows) to connect to your instance and ensure that everything is working correctly.

3. Consider that once a drill instance is launched, actual resources will be created in your AWS account and you will be billed for these resources. You can terminate the operation of launched recovery instances once you verify that they are working properly without impact for data replication.

4. We recommend you test as often as possible. Customers should test once a year at a minimum, even if it means reducing scope and testing a portion of the application or business function portfolio. This ensures the team is comfortable with the disaster recovery plan while also allowing them to identify any issues or required changes.

When preparing for disaster recovery test, it is critical to ensure that your Drill environment is configured properly. A drill will be conducted while the production environment remains intact. To minimize impact to the production environment, we recommend the following:

* **Network considerations**
    * **Subnet configuration**
        * **CIDR range**
            * You will want to ensure that your Drill subnet is configured with the same CIDR range size as your failover subnet. This will ensure that the subnets are sized properly and any IP adjustments to the Drill/Failover machines remains the same.
            * With this in mind, you will want to ensure that the subnet you are launching Drill instances is in an isolated network with no route to the source environment or production systems. This will ensure there are no IP address or routing conflicts during testing. We also recommend configuring security groups and access control lists to further reinforce these boundaries.
    * **Routing**
        * If your Drill requires access to services or dependencies outside the Drill subnet, you should ensure the appropriate routing policies and rules are configured in the Drill subnet to support this connectivity.
        * **Updating Launch Template to the Drill subnet**
            * By default, you will want to have the Launch Templates configured for your failover subnet. During a Drill, you will need to change that section of the Launch Template to the Drill subnet. You can find information on this process in EC2 launch template Additionally, Launch settings can be changed for a single server or for multiple servers through the Elastic Disaster Recovery console. This option allows you to quickly make changes to multiple servers at once. Refer to Configuring launch settings for more details on making bulk changes to your Launch Templates.
    * **Infrastructure Services (including Active Directory [AD], DNS, and more)**
        * Depending on the criteria for a successful Drill, you may need your Drill servers to connect to services such as AD or other infrastructure services to complete a Drill. This might require additional scripting (or usage of appropriate SSM documents to automate the usage of AD after launch).
            * With Elastic Disaster Recovery, you can replicate all applications and services, including AD. With this approach it is recommended to launch the Drill version of AD first and wait until the service is up and running. Once the service is up, you can launch the other applications or servers. This will ensure that the AD servers are ready to provide critical functions and services, such as authentication and authorization.
            * An alternative approach is to extend AD to the Drill subnet. We recommend working with your system administrators to define the best method for your use case.

Prior to launching a drill instance, ensure that your source servers are ready for testing by looking for the following indicators on the source servers page, as detailed below:

* Under the Ready for Recovery column, the server should show Ready. This means that the initial sync has completed, and all data from the source server has been replicated to AWS.
* Under the Data Replication Status column, the server should show the Healthy status, but you can also launch the source server if the system is undergoing Lag or even Stall. In that case, the data may not be up to date. You can still launch a drill instance from a previous point in time.
* Under the Pending Actions column, the server should show Initiative Recovery Drill if no drill instances have ever been launched for the server. Otherwise, the column will be blank. This helps you identify whether the server has had a recent drill launch.

### Launching drill instances

To launch a drill instance for a single source server or multiple source servers, go to the Source servers page and check the box to the left of each server for which you want to launch a drill instance.

1. Open the Initiate recovery job menu and select Initiate drill.

2. Select the PiT snapshot from which to launch the drill instance for the selected source server.

    a. You can either select the Use most recent data option to use the latest snapshot available or select an earlier specific Point-in-time snapshot. You may opt to select an earlier snapshot in case you wish to return to a specific server configuration before a disaster occurred. After you have selected the PiT snapshot, select Initiate drill.

3. The Elastic Disaster Recovery Console will indicate Recovery job is creating drill instance for X source servers when the drill has started.

4. Choose View job details on the dialog to view the specific Job for the test launch in the Recovery job history tab.

### Successful drill instance launch indicators

You can check that the Drill instance launch started successfully through several indicators on the Source servers page by following these steps:

* The Last recovery result column will show the status of the recovery launch and the time of the launch. A successful drill instance launch will show the Successful status. A launch that is still in progress will show the Pending status.
* The launched drill instance will also appear on the Recovery instances page.

## Invoking Recovery

To launch your recovery instances quickly, you should preconfigure how those instances are to be launched and perform drills in order to make sure all of your network and application settings are properly configured. You can configure how your instances will be launched by editing the Launch settings for each source server. Launch settings can be configured immediately when a source server has been added to Elastic Disaster Recovery–there is no need to wait for the initial sync process to finalize. Performing frequent drills is key for failover preparedness. Elastic Disaster Recovery makes it easy for you to launch drill instances as frequently as you want. Drills are non-disruptive and do not impact the source server or ongoing data replication. If you experience a disaster in the middle of a drill, you can launch a new recovery instance from the source server's current state or keep the instance you launched during the drill.

### Performing recovery

Prior to launching a Recovery instance, ensure that your source servers are ready for testing by looking for the following indicators on the Source servers page, as detailed below:

1. Configure your launch templates for each server you want to protect.

2. Under the Ready for recovery column, the server should show Ready. This means that initial sync has been completed and all data from the source server has been replicated to AWS.

3. Under the Data replication status column, the server should show the Healthy status, but you can also launch the source server if the system is undergoing Lag or even Stall. In that case the data may not be up to date. You can still launch a drill instance from a previous point in time.

4. Under the Pending actions column, the server should show Initiative recovery drill if no drill instances have ever been launched for the server. Otherwise, the column will be blank. This helps you identify whether the server has had a recent drill or recovery launch.

5. Under the Last recovery result column, there should be an indication of a successful drill or recovery instance launch sometime in the past. The column should state Successful and show when the last successful launch occurred. This column may be empty if a significant amount of time passed since your last drill instance launch.

To launch a recovery instance for a single source server or multiple source servers, go to the Source servers page, and check the box to the left of each server for which you want to launch a recovery instance, as detailed below:

1. Open the Initiate recovery job menu, and select Initiate recovery.

2. Select the PiT snapshot from which to launch the recovery instance for the selected source server.

    a. You can either select the Use most recent data option to use the latest snapshot available or select an earlier specific Point-in-time snapshot. You may opt to select an earlier snapshot in case you wish to return to a specific server configuration before a disaster occurred. After you have selected the PiT snapshot, choose Initiate recovery. You can find more information about PiT snapshots at Understanding Point in Time states.

3. The Elastic Disaster Recovery Console will indicate Recovery job is creating recovery instance for X source servers when the recovery has started.

4. Select View job details on the dialog to view the specific Job for the test launch in the Recovery job history tab.

Elastic Disaster Recovery is only one part of your disaster recovery plan. There are likely to be many other dependencies and services that will play a role in recovering from a disaster, and this should be factored in when conducting a drill or actual failover.

## Failback

Failback is the act of redirecting traffic from your recovery system and recovering data to your primary system. This is an operation that is performed outside of Elastic Disaster Recovery. Elastic Disaster Recovery assists you in performing the failback by ensuring that the state of your primary system is up to date with the state of your recovery system.

Before performing a failback, you want to make sure that any data that was written to your failover systems during the failover is replicated back to your original systems before you perform the actual failback and redirect traffic back to your primary systems. Elastic Disaster Recovery helps you prepare for failback by replicating the data from your Recovery instances on AWS back to your source servers with the aid of Elastic Disaster Recovery Mass Failback Automation client for on-premises vCenter servers.

**Note**: Failback is a planned downtime event and should ideally be scheduled during a maintenance window that minimizes disruption to users.

Elastic Disaster Recovery supports failback testing to confirm the process functions as expected. There are 2 types of testing available for failback:

1. Manual recovery instance/device mapping
2. Custom failback configuration file option which is the recommended approach

Failback testing ensures that you have a greater understanding of dependencies, planning, and requirements post disaster.

### Failback to on-premise vCenter using Elastic Disaster Recovery Mass Failback Automation client

Elastic Disaster Recovery allows you to perform a scalable failback for vCenter with the Elastic Disaster Recovery Mass Failback Automation Client (DRSFA Client). This allows you to perform a one-click or custom failback for multiple vCenter machines at once.

**Note**:
* The DRSFA client only works with vCenters source servers.
* The DRSFA client was only tested on vCenter versions 6.7 and 7.0.

### Failback Pre-requisites

1. **Recovery Server(s) in AWS**:

    a. Each server that is being failed back must have at least 4 GB of ram.
    
    b. The recovery instance used as a source for failback must have permissions to access Elastic Disaster Recovery via API calls. This is done using instance profile for the underlying EC2 instance. The instance profile must include the AWSElasticDisasterRecoveryRecoveryInstancePolicy. By default, the launch settings that Elastic Disaster Recovery creates for source servers already have an instance profile defined that includes that policy and that instance profile will be used when launching a Recovery Instance.
    
    c. Inbound port TCP 1500 must be open on the Recovery instance in AWS.

2. **Download Failback Client**:

    a. Download the latest version of the DRS Failback Client and upload it to your vCenter datastore. We always recommend using the latest version of the Elastic Disaster Recovery Failback Client.
    
    b. We recommend running SHA512 checksum verification on the Elastic Disaster Recovery Failback Client prior to using it with the DRSFA client. You can verify the checksum at the following address: https://aws-elastic-disaster-recovery-hashes-{REGION}.s3.amazonaws.com/latest/failback_livecd/aws-failback-livecd-64bit.iso.sha512

3. **DRSFA Client Server**:

    a. The server needs to be able to communicate with your vCenter environment.
    
    b. The server must have at least 4 GB of RAM.
    
    c. The server must run Python 3.9.4 with pip installed (other versions of Python will not work).
    
    d. The server requires some tools for DRSFA Client installation. The installer will attempt to install them if they are not already present.
    
        * See the requirements.txt file. These libraries will be installed automatically by DRSFA Client.
    
    e. The vCenter source servers must have two CD ROM devices with IDE controllers attached to run the DRSFA client - one for the Elastic Disaster Recovery Failback Client and one for the drs_failback_automation_seed.iso
    
        * If no attached CD ROM devices are found, the DRSFA client will attempt to add the CD ROM devices.

4. **VCenter**:

    a. Create a Vcenter User/Role
    
    b. You need to have the following vCenter API credentials and permissions: 'Virtual machine' : [ 'Change Settings', 'Guest operation queries', 'Guest operation program execution', 'Connect devices', 'Power off', 'Power on'. 'Add or remove device', 'Configure CD media] 'Datastore': ['Browse datastore']
    
    c. vCenter credentials should only be constrained to the VMs you plan to failback.
    ![vCenter User/Role Permissions Example](./assets/images/vcenter-permissions.png)
5. **AWS**:

    a. **Failback AWS credentials**
    
    * Performing a failback with the Elastic Disaster Recovery Failback Client requires AWS credentials with the AWSElasticDisasterRecoveryFailbackInstallationPolicy policy. You can create temporary credentials with AWS STS or create an IAM user and Access keys. As a best practice, using temporary security credentials (IAM roles) is recommended over creating long-term access keys. The credentials are only used during Failback Client installation.
        
        * **Generating IAM Credentials**: To generate the temporary credentials required to install the Elastic Disaster Recovery Failback Client, perform the following steps:
        
            * Create a new IAM Role with the AWSElasticDisasterRecoveryFailbackInstallationPolicy policy attached.
            * Request temporary security credentials via AWS STS using the AssumeRole API.
            * Learn more about using temporary security credentials with AWS CLI in the IAM Documentation.
    
    b. **CloudWatch Logging**:
    
    * It is recommended to create a CloudWatch Log group named DRS_Mass_Failback_Automation for tracking and managing the logs related to the DRSFA Client.
        
        * If the log group is not created or if it's named incorrectly, the DRSFA client will continue to function, but the logs won't be forwarded to CloudWatch. This could limit the ability to monitor and troubleshoot the DRSFA client's operations.
        
        * Refer to Create a log group in CloudWatch Logs for detailed instructions on how to work with log groups.

### DRSFA Client

**Note**:
* Prior to running the DRSFA Client, you must first install it. Installing the client is a one-time operation.
* The DRSFA client was fully tested on Ubuntu 20.04 and an installation script for this version is provided.

Once you are ready to perform a failback to your original source servers or to different servers (for example, if the original servers were lost), perform the following steps:

1. **Setup Ubuntu 20.04 ISO for the DRSFA Client**: The DRSFA client can be configured in either AWS or the on-premise vCenter environment:
    
    a. **AWS**:
    * Ensure network connectivity between the DRSFA Client's subnet and your vCenter environment.
    * Follow the steps for launching an EC2 instance, but select Ubuntu Server 20.04 LTS (HVM) AMI instead of Amazon Linux AMI. Proceed with instance configuration according to documentation.
    
    b. **vCenter**:
        
    * Download the Ubuntu Server 20.04 LTS image ISO from the Ubuntu download site.
    * Upload the ISO to the desired VMware Datastore.
    * Create a New Virtual Machine in your vCenter environment for the DRSFA server using the uploaded ISO.

2. **Installing the DRSFA Client**:
    
    a. Once your VM instance is set up and ready, connect to the Ubuntu instance and download the DRSFA client using the following command:
     
        wget https://drsfa-us-west-2.s3.us-west-2.amazonaws.com/drs_failback_automation_installer.sh
    
    b. Execute the installation script by running the following command:
        
        bash drs_failback_automation_installer.sh`
    
    c. Apply the new changes made to your profile configuration in the previous step, reload it by running the following command:
        
       source ~/.profile
       
    *  **Note**: This command may ask for a sudo password if you use the Ubuntu ISO. Enter the password but do not run this command as sudo.
    
    d. Once installation is complete, you will need to set up a password for the VM on which the DRSFA client is ran. This is done by generating a seed.iso file that you must upload to your Datastore. This can be done by running the following command:
        
        bash drs_failback_automation_seed_creator.sh
    
    e. Two files will be generated, the drs_failback_automation_seed.iso file and the drs_failback_automation_seed.iso.sha512 hash.
    
    f. Upload the seed.iso file to the same Datastore where the DRS Failback Client ISO file is stored.


3. **Running the DRSFA Client**:
    
    a. Navigate to the drs_failback_automation_client directory to run the DRSFA client:
        
        cd drs_failback_automation_client
    
    b. Enter the following parameters in a single line or settings the environment variables one by one, replace the defaults with your specific parameters and paths followed by the python drs_failback_automation_init.pyc command and press enter
    
    c. **Parameters**:
        
    * AWS_REGION=XXXXX – The AWS Region in which your Recovery instances are located.
    * AWS_ACCESS_KEY=XXXXX – The AWS Access Key you generated for the DRSFA client.
    * AWS_SECRET_ACCESS_KEY=XXXXXX – The AWS Secret Access Key you generated for the DRSFA client.
    * DRS_FAILBACK_CLIENT_PASSWORD=XXXXXX – The custom password you set for the Failback Client in the drs_failback_automation_seed.iso file.
    * VCENTER_HOST=XX.XX.XXX.XXX – The IP address of the vCenter Host.
    * VCENTER_PORT=XXX – The vCenter Port (usually 443)
    * VCENTER_USER=sample@vsphere.local – The vCenter username
    * VCENTER_PASSWORD=samplepassword – The vCenter password
    * VCENTER_DATASTORE=DatastoreX – The Datastore within vCenter where the Failback Client ISO file (aws-failback-livecd-64bit.iso) and seed.iso file (drs_failback_automation_seed.iso) are stored.
    * VCENTER_FAILBACK_CLIENT_PATH='samplepath/aws-failback-livecd-64bit.iso' – Failback Client ISO path in the Datastore.
    * VCENTER_SEED_ISO_PATH='samplepath/drs_failback_automation_seed.iso' – The seed.iso file path in the Datastore.

    ![DRSFA Client Parameters Configuration Example](./assets/images/drsfa-parameters.png)

     **Note**:
    
    * SSL verification is active by default. If you want to deactivate SSL verification, then add the following parameter: DISABLE_SSL_VERIFICATION=true
        
    * By default, the DRSFA client will initiate a failback for 10 servers at once (if failing back more than 10 servers). However, if you would like to change the default value, use the THREAD_POOL_SIZE parameter.
    
    d. **Example**:
    
       AWS_REGION="us-east-1" AWS_ACCESS_KEY="AKIAIOSFODNN7EXAMPLE" AWS_SECRET_ACCESS_KEY="wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY" DRS_FAILBACK_CLIENT_PASSWORD="lab123" VCENTER_HOST=172.16.1.196 VCENTER_PORT=443 VCENTER_USER="drsfa-user@vsphere.local" VCENTER_PASSWORD="@dmin123" DISABLE_SSL_VERIFICATION=true VCENTER_DATASTORE="WorkloadDatastore" VCENTER_FAILBACK_CLIENT_PATH="drsfa-lab/aws-failback-livecd-64bit.iso" VCENTER_SEED_ISO_PATH="drsfa-lab/drs_failback_automation_seed.iso" python3 drs_failback_automation_init.pyc

    ![DRSFA Client Command Example](./assets/images/drsfa-command-example.png)
        ```
   
    e. Once the client has connected successfully and finished verification, you will be presented with the below options under a "What would you like to do?" menu:
    
        1. One-click Failback
        2. Perform a custom Failback
        3. Generate a default failback configuration file
        4. Find servers in vCenter
        5. Help
        6. Exit
    ![DRSFA Client Menu Options](./assets/images/drsfa-menu-options.png)
    
    f. If you select One-Click Failback option:
    * Enter a custom prefix for the results output for this failback operation. This file will be saved in the /drs_failback_automation_client/results/Failback directory.
        
             Example: drsfa_output
        
        * Note: If failback replication has already been started for some of the Recovery instances, the console will prompt you whether you want to skip the instances that are already in failback or restart replication for those instances.
        
    * The DRSFA client will list the Recovery instances that are currently present in your AWS Account. The client will then prompt you "Would you like to continue?". Enter Y to continue.
        
    * The client will initiate failback. You can see the failback progress on the Recovery instances page in the Elastic Disaster Recovery Console.
        
    * On each failback VM console you can see that Failback has been initiated by the DRSFA client.
        
    * Once the failback has been complete, the DRSFA client will display the results of the failback, including the number of servers for which replication has successfully been initiated and the number of servers for which the failback operation failed.
        
    * After the Job is complete, you will see the VM reboot back to the recovered OS.
    
    g. If you select Perform a Custom Failback option:
    
    * The custom failback option gives you more control and flexibility over the failback process. When utilizing the custom failback option, you will first create a failback configuration file, in which you can edit specific settings for each individual machine, and you will then use this file to perform a failback.
        
    * To use the custom failback option, you can first generate a default failback configuration file through the client. Once the client has connected successfully and finished verification, select the "Generate a default failback configuration file" option under "What would you like to do?" menu.
        
    * Enter a custom prefix for the configuration file name. The configuration file will be created as a JSON file in the /drs_failback_automation_client/Configurations/ folder with the following name: "{prefix}_{account_id}_{region}.json"
        
    * You can edit any of the fields in the file in order to correctly configure it. The file will display the following fields for each server. You can edit every field to have absolute control over your failback configuration for each machine. Ensure to save your changes.
        
            * NETMASK
            * VCENTER_MACHINE_UUID
            * PROXY
            * DNS
            * CONFIG_NETWORK
            * IPADDR
            * GATEWAY
            * SOURCE_SERVER_ID
            * DEVICE_MAPPING
        
        * Option 1: You can use option 3 to generate the configuration file which will auto populate with all the details.
    
          ![DRSFA Client Configuration File Example](./assets/images/drsfa-config-file.png)

          Example config file when failing back to an existing sever in Vcenter:
          
          ![DRSFA Custom Failback Configuration Selection](./assets/images/drsfa-custom-failback-selection.png)        

        * Option 2: If you need to failback to a new server in vCenter you need to use Static Network settings or want to modify device mapping - you will need to generate a configuration file and map those details manually. Example:

        ```json
        {
          "CONFIG_NETWORK": "STATIC",
          "GATEWAY": "10.0.0.1",
          "NETMASK": "255.255.255.0",
          "IPADDR": "10.0.0.200",
          "DNS": "10.0.0.1",
          "RECOVERY_INSTANCE_ID": "i-1234567890abcdefg",
          "VCENTER_MACHINE_UUID": "42301234-1111-2222-3333-54321example",
          "VCENTER_SOURCE_SERVER_MACHINE_UUID": "42301234-1111-2222-3333-54321example",
          "PROXY": "",
          "BLOCK_DEVICE_MAPPING": [
            {
              "mappedDevice": "/dev/sda",
              "displayName": "",
              "recoveryVolumeName": "",
              "previousSizeBytes": "",
              "recoveryInstanceDevice": "/dev/nvme0n1",
              "sourceServerDevice": "/dev/sda1"
            },
            {
              "mappedDevice": "/dev/sdb",
              "displayName": "",
              "recoveryVolumeName": "",
              "previousSizeBytes": "",
              "recoveryInstanceDevice": "/dev/nvme1n1",
              "sourceServerDevice": "/dev/sdb"
            }
          ]
        }
        ```
        * Once the file is generated you run select option 2 and specify the path to the config file.

        ![DRSFA Custom Failback Configuration Selection](./assets/images/drsfa-custom-path-selection.png)        
    
    h.  The CONFIG_NETWORK value should be set to "DHCP" if you are using DHCP. The value should be set to "STATIC" if you want to manually configure the network settings. 
     * If CONFIG_NETWORK is set to "DHCP", then the DNS, IPADDR, GATEWAY, NETMASK, and PROXY parameters are ignored but should not be deleted.
    * If you are using a proxy server, leave the PROXY field as an empty string, do not remove it.
        
    * If a source server does not have an attached recovery instance, the file will still be generated, but the SOURCE SERVER ID field will be empty.
        
    
    i. Once you are done editing your configuration file, rerun the DRSFA client and select the "Perform a Custom Failback" option.
        
    * Select your configuration file. You can either define a custom path or select the default path that's automatically displayed by the client.
        
    * Enter a custom prefix for the results output for this failback operation. This file will be saved in the /drs_failback_automation_client/results/Failback directory.
        
            * Example: drsfa_output
        
    * Note: If failback replication has already been started for some of the Recovery instances, the console will prompt you whether you want to skip the instances that are already in failback or restart replication for those instances.
        
    * The DRSFA client will list the Recovery instances that are currently present in your AWS Account. The client will then prompt you "Would you like to continue?". Enter Y to continue.
        
    * The client will initiate failback. You can see the failback progress on the Recovery instances page in the Elastic Disaster Recovery Console.
        
    * On each failback VM console you can see that Failback has been initiated by the DRSFA client.
        
    * Once the failback has been complete, the DRSFA client will display the results of the failback, including the number of servers for which replication has successfully been initiated and the number of servers for which the failback operation failed.
        
    * After the Job is complete, you will see the VM reboot back to the recovered OS.
    
    j. If you choose Find servers in vCenter option:
    
    * Select the Find servers in vCenter option to find machines in vCenter. This makes it easier to discover the disks/volumes of your machines for custom failback.
        
    * Enter a name to filter or press Enter to see all results. Choose Yes to print your results.
        
    * The results will be exported to the /drs_failback_automation_client/results/VMFinder folder. The results will be named after the vCenter IP and the time stamp "{vcenter_host}_{ts}.txt"
        
    * The following will be displayed for each server:
        
        * Name
        * UUID
        * Disk and volume info

## Troubleshooting

Follow the steps below for troubleshooting:

1. The full results of the failback will be exported as a JSON file to the failback client folder.

    a. /drs_failback_automation_client/results/Failback folder with the custom prefix you set, the AWS account ID, the AWS Region, and a timestamp.
    
    b. The JSON file will display the following:
    
    * The AWS ID of the recovery instance
    * The status of the failback (succeeded, skipped, or failed)
    * A message (which provides the cause for failure in the case of failure)
    * The vCenter VM UUID

2. If failback failed for any of your machines, you can troubleshoot the failure by looking at the machine configuration failback_hosts_settings.json file in the same folder.

3. On the DRSFA Client, there are two logs available under /drs_failback_automation_client/:

    a. 'drs_failback_automation.log' displays any data that would typically have logged to the VM console.

4. To find the log for a specific server, open the VM, and find the drs_failback_automation.log and failback.log file, which can be used for troubleshooting.

    a. 'failback.log' deals with logged errors from the failback client.
    
    b. 'drs_failback_automation.log' displays any data that would typically have logged to the VM console.

5. Common Errors:

    a. Failed to connect to vCenter host: ("Failed to connect to the vCenter host. Make sure the host's parameters and credentials are correct and that connectivity is allowed. Then, try again")
    
    *  Verify vCenter credentials
    * Verify vCenter permissions
    * Ensure correct value is set for DISABLE_SSL_VERIFICATION environmental variable
    
    b. "Unexpected vCenter error: Failed to authenticate with the Guest OS using Supplied Credentials"
    
    * Ensure target failback VM boots to [Enter AWS region to failback from]
    * Check for any boot related failures on the target failback VM in /var/logs/boot.log (verify networking status)
    * If Static network configuration was set for failback, review /drs_failback_automation.log and /failback.log if available.
    
    c. Unexpected Error: "Failed to search in Datastore. An unexpected error occurred. Please try downloading the DRS Mass Failback Automation Client again or contact AWS support" or Unexpected Error: "Failed to search in datastore"
    
    * Ensure correct iso/datastore path is provided in the environmental variables (without datastore name, just the path).
    * Check the logs under /drs_failback_automation_client/Results/Failback

For more information, review the following resources: [Performing a failback with the Elastic Disaster Recovery Mass Failback Automation client](https://docs.aws.amazon.com/drs/latest/userguide/failback-mass-automation.html).

## Conclusion

This implementation guide provides guidance and best practices to deploy Elastic Disaster Recovery Service for on-premises applications.

In this guide, we detailed how to think about design patterns when provisioning resources like the staging area subnets, recovery subnets, EBS volumes, and other components of the Elastic Disaster Recovery architecture. The guide also provides the required background to understand the core principles and technical know-how required to follow the concepts behind various stages of the Elastic Disaster Recovery implementation process, ensuring that the resources you provision remain robust, even when facing unforeseen obstacles. We also covered observability practices to help you leverage the right set of tools and services, like replication health monitoring and cost optimization alerts, so that your insights drive proactive action.

The guide also highlighted the importance of testing your disaster recovery strategy by using drills. We discussed key considerations when performing an actual failover in response to a real disaster and how you could ultimately restore to normal operations using the failback functionality. We hope this guide will serve as a foundation for using Elastic Disaster Recovery across the entire disaster recovery lifecycle so that your business can respond to unexpected impairments.

## Notices

Copyright Amazon.com, Inc. or its affiliates. All Rights Reserved.

Permission is hereby granted, free of charge, to any person obtaining a copy of
this software and associated documentation files (the "Software"), to deal in
the Software without restriction, including without limitation the rights to
use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of
the Software, and to permit persons to whom the Software is furnished to do so.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS
FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR
COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER
IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN
CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.

## Authors

1. Ashutosh Agarwal
2. Ben Lipman
3. Jason Perry
4. Priyam Reddy
5. Sravan Rachiraju