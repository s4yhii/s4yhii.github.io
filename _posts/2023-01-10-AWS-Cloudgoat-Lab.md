---
title: Cloudgoat rce_web_app scenario
date: 2023-01-10 12:00:00 -0400
categories: [Web Security, AWS]
tags: [web security, labs, owasp, rce, cloud, aws]
---

# Cloudgoat RCE_WEB_APP Scenario

## Introduction
CloudGoat is a training and learning platform developed by Rhino Security Labs to help individuals and organizations understand the risks and vulnerabilities associated with cloud-based applications. One of the scenarios available on CloudGoat is the RCE_web_app scenario, which allows users to practice exploiting remote code execution vulnerabilities in a web application running on the cloud.

In this blog post, we will walk through the RCE_web_app scenario in CloudGoat and provide a step-by-step guide on how to exploit the vulnerability and gain access to the application's backend. We will also discuss the significance of this vulnerability and how it can be prevented in real-world scenarios. By the end of this post, you should have a better understanding of the risks and challenges associated with web application security in the cloud and how to mitigate them. So, let's get started!

## Solution

Once we've done all the setup process we will start with the explotation 
There are two attack paths in this scenario, Lara and McDuck. The two paths start in different places but converge at a certain point. We will start with the Lara path, but you can skip ahead to the McDuck path here.

### Configuring the Lara Attack Path
When the build completes, some information will be displayed to get you started. For the “rce_web_app” scenario, you are given two sets of keys. Create AWS CLI profiles for the given keys using the command below.

```bash
aws configure --profile Lara
```

The first step in any pentest is figuring out what we already have access to–usually by confirming permissions. Unfortunately, Lara isn’t authorized to perform many of the calls typically used to confirm permissions. Because of this, we will have to manually look around to find what she has access to.

inding EC2 Instances
We decided to start by finding out which EC2 instances Lara has access to by running the following command.

aws ec2 describe-instances --profile Lara
The output from this command shows that Lara has access to an EC2 instance with a public IP. However, when we try to navigate to the IP in a browser, we get a timeout. We can conclude that this might be worth looking at deeper later, but for now, we will keep looking around.

Finding Load Balancers
Next, we will take a look at any load balancers we may have access to by running the following command.

aws elbv2 describe-load-balancers --profile Lara
We see a publicly accessible load balancer in the output from this command. When we try to navigate to it in the browser, we are greeted by a message from a “Gold-Star Executive Interstellar Travel Rewards” program that references a “secret” URL. The image below shows this landing page.

Landing Page
Discovering and Listing S3 Buckets
Since there’s nothing we can do with this web app yet, listing S3 buckets is a good next step. Running the following command shows us the buckets we can list.

aws s3 ls --profile Lara
The output of this command shows us that Lara can list 3 buckets, as pictured below.

Lara CloudGoat
By running the following command, we will be able to find which buckets we are able to list, and what the contents of them are.

aws s3 ls s3://<bucket> --recursive --profile Lara
We got an “Access Denied” message when running this command on two of the buckets, but we were able to get a list of the contents of one bucket. This bucket appears to contain logs for the load balancer we discovered earlier, which we can download to inspect for useful information. Run the following command to download the log file to your working directory.

aws s3 cp s3://<bucket>/cg-lb-logs/AWSLogs/793950739751/elasticloadbalancing/us-east-1/2019/06/19/555555555555_elasticloadbalancing_us-east-1_app.cg-lb-cgidp347lhz47g.d36d4f13b73c2fe7_20190618T2140Z_10.10.10.100_5m9btchz.log . --profile Lara
Opening the downloaded file in a text editor, we can see that we are looking at a log file for a load balancer, though the instance ID differs from the load balancer we currently have access to. Inspecting the file for useful information turns up a log entry that reveals an HTML page present on the domain. The image below shows the log entry for the HTML page that could be useful.

Log Entry
Going to the exact URL in the log doesn’t turn up anything–the page doesn’t load. Thinking back to the “Gold-Star Executive Interstellar Travel Rewards” program webpage that we found earlier, maybe this is part of the “secret” URL. When we append the discovered HTML page to the DNSName or public IP of the load balancer we found earlier, we see the following webpage, which looks like a great candidate for trying some command injection.

Command Injection
Exploiting the Web App
Upon finding the web app, we try a few commands and discover that it is vulnerable to remote command execution. We can then run the following command to see who the current user is.

whoami
To our delight (or horror), running the “whoami” command reveals that the web application is running as root, so every command we run via the RCE is also being run as root. 

By using the following command, we can see the public IP of the instance the app is running on.

curl ifconfig.co
The IP that gets returned is the same IP as the EC2 instance we found earlier. To take a look at that instance again, we run the following command.

aws ec2 describe-instances --profile Lara
Pictured below is the “SecurityGroups” entry for the instance, showing that it is a member of the “cg-ec2-ssh-cgid<uniqueID>” group. This tells us that SSH access is very likely enabled.

SSH Access
Adding SSH Public Key
Next, we want to see what would happen if we simply add our SSH certificate to the authorized_keys file. To do this, we first need to generate some SSH keys. Running the following command on your local machine will generate public and private SSH keys.

ssh-keygen
Attempting to SSH in as root just displays a message advising you to use the ubuntu user. So, we will echo the public key we just generated to the authorized_keys file of the ubuntu user by running the following command via the RCE.

echo "<publicKey>" >> /home/ubuntu/.ssh/authorized_keys
At this point, we are going to shift focus briefly to cover the other attack path in this scenario, McDuck. Lara and McDuck start in different places but their paths converge at a certain point, which you can skip ahead to here.

McDuck Attack Path
Setting Up the McDuck Attack Path
This walkthrough will use the McDuck keys that were generated when we deployed the scenario. Run the below command and follow the prompts to configure a McDuck profile in the AWS CLI.

aws configure --profile McDuck
Discovering SSH Keys
We always want to start a pentest by seeing what we have access to. In the Lara attack path, there were several S3 buckets that proved useful, so that seems like a good place to start. The following command lists the S3 buckets available to McDuck.

aws s3 ls --profile McDuck
As with Lara, we can list 3 buckets, but when we try to view the contents of the “cg-logs” and “cg-secret” buckets we get an “Access Denied” message. Running the following command will list the contents of the “cg-keystore” bucket.

aws s3 ls s3://cg-logs-s3-bucket-cgid<uniqueID> --recursive --profile McDuck
The image below displays the output from the previous command showing what appear to be SSH keys in the “cg-keystore” bucket.

cg-keystore bucket
The next step is to download these files and see what else we can find. Run the following commands on your local machine to create a directory for the cloudgoat and cloudgoat.pub files and download them to potentially use later.

mkdir mcduck && cd mcduck 
aws s3 cp s3://cg-keystore-s3-bucket-cgid<uniqueID>/cloudgoat . --profile McDuck
aws s3 cp s3://cg-keystore-s3-bucket-cgid<uniqueID>/cloudgoat.pub . --profile McDuck
Now that we’ve looked at S3, we should take a look at EC2. Run the following command to see if there are any EC2 instances McDuck can list.

aws ec2 describe-instances --profile McDuck
We see that we can list a single EC2 instance that has a public IP and is a member of a group that indicates SSH access might be possible. The image below shows the “Groups” section of the instance, with an entry for “cg-ec2-ssh-cgid<uniqueID>”.

“cg-ec2-ssh-cgid” entry
Final Attack
As stated earlier, the Lara and McDuck attack paths merge starting here. So, using either the SSH private key you generated as Lara or the SSH private key discovered by McDuck, attempt to connect to the public IP of the instances via SSH, passing in your private key file.

ssh -i <private_key> ubuntu@107.22.55.92
Exploiting the EC2 Instance
We now have remote root access to the EC2 instance. After looking around the machine a bit there doesn’t seem to be much of interest locally. The next step would be to see what access the EC2 instance has. To do this, we need to install the AWS CLI on the EC2 instance and then repeat the enumeration we did earlier. Run the following command to install the AWS CLI.

sudo apt-get install awscli
We don’t need to configure a profile in this case because the AWS CLI will automatically use the keys provided by the EC2 MetaData service. Running the following command will show what buckets the EC2 instance can access.

aws s3 ls
We see the same buckets as we did before, but this time we can access the “cg-secret-s3” bucket. Run the following command to see what the bucket contains.

aws s3 ls s3://cg-secret-s3-bucket-cgid<uniqueID> --recursive
Copy the discovered file to your working directory on the EC2 instance and inspect the contents by running the following command.

aws s3 cp s3://cg-secret-s3-bucket-cgidzay5e3vg5r/db.txt . | cat db.txt
This outputs database credentials. Score! The next step is to find where to use them. To see if we have any DB instances running, we can execute the following command.

aws rds describe-db-instances --region us-east-1
The output of this command is pictured below.

Address
Connecting to the Database
With the location of an RDS database confirmed (which just happens to have the same MasterUsername and DBName as our discovered credentials), we attempt to connect to the database. Run the following command in order to do this.

psql postgresql://cgadmin:Purplepwny2029@<rds-instance>:5432/cloudgoat
Note: an alternative path exists to discover the RDS Instance. Simply using curl to read the user_data from the instance metadata will reveal the RDS instance location and credentials.

curl http://169.254.169.254/latest/user-data
Now that we’ve connected to the database, the final step is to list the tables and discover the flag. Running the following commands within psql, when connected to the RDS instance, will list the tables, display the flag, and complete this scenario.

\dt
select * from sensitive_information
The “Super-secret-passcode” is your flag, as pictured below. Congratulations, you’ve now completed the rce_web_app scenario! This was not an easy scenario, and neither were the attack paths.

Flag
Conclusion
This walkthrough demonstrated both the Lara and McDuck attack paths for the “rce_web_app” scenario. In both cases, recursive reconnaissance, or reexamining your permissions and environment when a new level of access has been achieved, yielded useful information that ultimately led to the discovery of the victory flag. Attackers following the Lara path leveraged web application pentesting skills to add themselves to the SSH “authorized_keys” file. Once connected to the EC2 instance, attackers reevaluated their permissions to discover new information in a previously inaccessible S3 bucket or queried the EC2 metadata service to discover an RDS instance. Finally, the attacker connected to the RDS instance and discovered the victory flag.

![](https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/htb/challenges/ch1.jpg)

