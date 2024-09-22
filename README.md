# AWS Three-Tier Web Architecture

## Overview:

This hands-on guide walks you through setting up a three-tier web architecture in AWS. You'll manually create and configure the necessary network, security, application, and database components to deploy this architecture in a scalable and highly available manner.

## Target Audience:

This guide is for individuals in technical roles who have foundational knowledge of AWS services such as VPC, EC2, RDS, S3, ELB, and the AWS Management Console.

## Requirements:

1. An active AWS account. If you don’t have one, follow the instructions [here](https://aws.amazon.com/console/) and click “Create an AWS Account” at the top right corner.
2. An IDE or text editor of your choice.

## Architecture Summary

![Architecture Diagram](/demos/3TierArch.png)

In this architecture, an Application Load Balancer forwards client requests to EC2 instances in the web tier. These instances, running Nginx, serve a React.js website and route API requests to the application tier's internal load balancer. The internal load balancer forwards these requests to Node.js application servers that interact with a multi-AZ Aurora MySQL database. Each tier is configured with load balancing, health checks, and auto-scaling to ensure availability.

## Initial Setup - Part 0

### Clone the Repository

```bash
gh repo clone ayuspoudel/AWS-Three-Tier-Web-Architecture
```

### Create an S3 Bucket

1. Open the S3 service in the AWS console and create a new bucket.
2. Choose your desired region and assign a unique name to the bucket.

### Set Up IAM Role for EC2

1. Open the IAM dashboard in the AWS console and create a new role for EC2.
2. Select EC2 as the trusted entity.
3. Attach the following AWS managed policies to allow the instances to retrieve code from S3 and use Systems Manager Session Manager for secure, keyless connections:
   - AmazonSSMManagedInstanceCore
   - AmazonS3ReadOnlyAccess
4. Name your role and create it.

## Networking and Security Configuration - Part 1

We will now create the necessary VPC networking components and security groups to protect our EC2 instances, Aurora database, and Elastic Load Balancers.

### VPC and Subnet Creation

#### VPC Setup

1. Navigate to the VPC dashboard in the AWS console and go to <b>Your VPCs</b>.
2. Select VPC only and configure the settings with a Name tag and a CIDR block of your choice.

   <b>Note:</b> Ensure that you consistently deploy all resources in the same region.

   <b>Note:</b> Choose a CIDR block that supports the creation of at least 6 subnets.

   ![VPC](/demos/FillVPCSettings.png)

#### Creating Subnets

1. Create subnets by navigating to Subnets on the left side of the dashboard and clicking Create Subnet.
2. Create six subnets across two availability zones—three subnets per availability zone, each corresponding to one layer of our three-tier architecture.

   <b>Note:</b> Use a naming convention that helps you remember each subnet’s purpose, such as Public-Web-Subnet-AZ-1, Private-App-Subnet-AZ-1, Private-DB-Subnet-AZ-1.

   <b>Note:</b> Ensure your subnet CIDR blocks are subsets of your VPC CIDR block.

   Your subnet configuration should resemble the one shown below. Ensure you have three subnets across two availability zones.

   ![subnet](/demos/FillSubnetDetails.png)

### Configuring Internet Connectivity

#### Internet Gateway Setup

1. To enable internet access for public subnets, create and attach an Internet Gateway. Go to Internet Gateway on the Dashboard.
2. Name the Internet Gateway and create it.
3. Attach the Internet Gateway to your VPC via the success message or the Actions drop-down.

   ![internetGateway](/demos/AttachIGW1.png)

4. Select the correct VPC and attach the gateway.

#### NAT Gateway Setup

1. To allow instances in the private app subnets to access the internet, create a NAT Gateway in each public subnet. Go to NAT Gateways on the left side of the dashboard and click Create NAT Gateway.
2. Provide a name, select a public subnet, and allocate an Elastic IP. Click Create NAT Gateway.

   ![NAT](/demos/FillNATGWDetails.png)

3. Repeat these steps for the second public subnet.

### Routing Configuration

1. Navigate to Route Tables on the left side of the VPC dashboard and click Create Route Table. First, create a route table for the public web subnets and name it appropriately.

   ![](/demos/CreatePublicRT.png)

2. After creating the route table, edit the routes by navigating to the Routes tab and clicking Edit routes.

   ![](/demos/EditRoutes.png)

3. Add a route that directs traffic from the VPC to the Internet Gateway for IPs outside the VPC CIDR block. Save the changes.

   ![](/demos/AddIGWRoute.png)

4. Edit the Explicit Subnet Associations of the route table by navigating to the route table details. Select Subnet Associations and click Edit subnet associations.

   ![](/demos/EditSubnetAssociations.png)

   Associate the route table with the public web subnets you created and save the associations.

   ![](/demos/AssociatePublicSubnets.png)

5. Next, create two more route tables—one for each private app subnet in each availability zone. These tables will route traffic destined for outside the VPC to the NAT Gateway in the corresponding availability zone.

   ![](/demos/PrivateSubnetRT1.png)

   After creating the route tables and adding routes, associate them with the appropriate private app subnets.

   ![](/demos/PrivateSubnetAssociation.png)

### Security Groups

1. Create security groups to control traffic to and from our Elastic Load Balancers and EC2 instances. Navigate to Security Groups under Security on the left side of the VPC dashboard.

2. First, create a security group for the public, internet-facing load balancer. Add an inbound rule allowing HTTP traffic from your IP.

   ![](/demos/ExternalLBSG.png)

3. Create a security group for the public instances in the web tier. Add an inbound rule that allows HTTP traffic from the security group you created for the load balancer. Also, add a rule to allow HTTP traffic from your IP for testing.

   ![](/demos/WebTierSG.png)

4. Create a security group for the internal load balancer. Add an inbound rule that allows HTTP traffic from the web tier instance security group.

   ![](/demos/InternalLBSG.png)

5. Create a security group for the private instances. Add an inbound rule allowing TCP traffic on port 4000 from the internal load balancer security group. This allows the internal load balancer to forward traffic to the private instances. Add another rule for port 4000 that allows traffic from your IP for testing.

   ![](/demos/PrivateInstanceSG.png)

6. Finally, create a security group for the private database instances. Add an inbound rule allowing traffic from the private instance security group to the MySQL/Aurora port (3306).

   ![](/demos/DBSG.png)

## Database Setup - Part 2

Deploy the database layer by creating Subnet Groups and a Multi-AZ Database.

### Creating Subnet Groups

1. Open the RDS dashboard in the AWS console and select Subnet Groups on the left. Click Create DB Subnet Group.
2. Name the subnet group, provide a description, and select the VPC we created.

   ![](/demos/FillSubnetGroupDetails1.png)

3. Add the subnets we created for the database layer in each availability zone. You may need to check the subnet IDs in the VPC dashboard to ensure accuracy.

   ![](/demos/FillSubnetGroupDetails2.png)

### Deploying a Multi-AZ Database

1. Navigate to Databases on the left side of the RDS dashboard and click Create Database.

2. Choose Standard create for a MySQL-Compatible Amazon Aurora database, and leave the Engine options as default.

   ![](/demos/DBConfig1.png)

3. Under Templates, select Dev/Test since this is not a production environment. Set a username and password for database access, and note them down.

   ![](/demos/DBConfig2.png)

4. In Availability and Durability, opt to create an Aurora Replica or reader node in a different availability zone. Under Connectivity, select the VPC, choose the subnet group created earlier, and disable public access.

   ![](/demos/DBConfig3.png)

5. Assign the security group we created for the database layer, ensure password authentication is selected, and create the database.

   ![](/demos/DBConfig4.png)

6. Once the database is provisioned, you will see a reader and writer instance in the database subnets of each availability zone. Note down the writer endpoint for later use.

   ![](/demos/DBEndpoint.png)

## Deploying the App Tier Instance - Part 3

Next, we'll create an EC2 instance for the application layer, configure the necessary software stack, and set up the database with the required schema and initial data.

### Deploying the App Instance

1. Go to the EC2 service dashboard and click Instances on the left. Click Launch Instances.

2. Select the Amazon Linux 2

AMI and the free tier eligible **T2.micro** instance type. Proceed to the next step.

3. In the Configure Instance Details page, ensure you select the correct Network, subnet, and IAM role. Use one of the private subnets created for the app layer.

   ![](/demos/ConfigureInstanceDetails.png)

4. Use the default settings for storage and tagging. In the Security Group section, select the security group for the private app layer instances created earlier. Then click Review and Launch. Ignore any warnings about port 22 since we're using Systems Manager Session Manager for connections.

5. After reviewing the configuration, click Launch. Proceed without a key pair since we’re using Systems Manager Session Manager.

### Connecting to the Instance

1. Once the instance is running, select it and click Connect at the top right corner. Choose the Session Manager tab and click Connect to open a new browser terminal.

   <b>Note:</b> If you encounter connection issues, check your network configuration and ensure the EC2 instance can route to the NAT gateways.

2. Upon connection, switch from the default `ssm-user` to `ec2-user` by executing the following command:

   ```bash
   sudo -su ec2-user
   ```

3. Confirm internet connectivity by pinging Google DNS:

   ```bash
   ping 8.8.8.8
   ```

   Stop the transmission by pressing Ctrl+C.

   <b>Note:</b> If you can't reach the internet, verify your route tables and subnet associations.

### Configuring the Database

1. Install the MySQL CLI:

   ```bash
   sudo yum install mysql -y
   ```

2. Connect to the Aurora RDS database using the writer endpoint:

   ```bash
   mysql -h YOUR-RDS-ENDPOINT -u YOUR-USER-NAME -p
   ```

3. When prompted, enter your password. After successful authentication, you'll be connected to the database.

   <b>Note:</b> If the connection fails, check your credentials and security groups.

4. Create a database called `webappdb`:

   ```bash
   CREATE DATABASE webappdb;
   ```

   Verify the creation with:

   ```bash
   SHOW DATABASES;
   ```

5. Create a `transactions` table within `webappdb`:

   ```bash
   USE webappdb;
   CREATE TABLE IF NOT EXISTS transactions(id INT NOT NULL AUTO_INCREMENT, amount DECIMAL(10,2), description VARCHAR(100), PRIMARY KEY(id));
   ```

   Verify table creation with:

   ```bash
   SHOW TABLES;
   ```

6. Insert sample data for testing:

   ```bash
   INSERT INTO transactions (amount,description) VALUES ('400','groceries');
   ```

   Verify the data insertion with:

   ```bash
   SELECT * FROM transactions;
   ```

7. Exit the MySQL client:
   ```bash
   exit
   ```

### Configuring the App Instance

1. Update the database credentials in the `application-code/app-tier/DbConfig.js` file from the GitHub repository. Fill in the necessary fields for hostname, user, password, and database with the credentials you configured earlier.

   <b>Note:</b> This is for simplicity in this lab; for production, consider using AWS Secrets Manager for storing credentials.

2. Upload the `app-tier` folder to the S3 bucket you created earlier.

3. In your SSM session, install the Node Version Manager (NVM) and Node.js:

   ```bash
   curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.38.0/install.sh | bash
   source ~/.bashrc
   nvm install 16
   nvm use 16
   ```

4. Install the PM2 process manager to keep the Node.js app running:

   ```bash
   npm install -g pm2
   ```

5. Download the `app-tier` code from your S3 bucket onto the instance:

   ```bash
   cd ~/
   aws s3 cp s3://BUCKET_NAME/app-tier/ app-tier --recursive
   ```

6. Navigate to the app directory, install dependencies, and start the app with PM2:

   ```bash
   cd ~/app-tier
   npm install
   pm2 start index.js
   ```

   Verify that the app is running correctly with:

   ```bash
   pm2 list
   ```

   If there are issues, check the logs:

   ```bash
   pm2 logs
   ```

7. Set PM2 to restart the app automatically on instance reboot:

   ```bash
   pm2 startup
   ```

   Copy and paste the command provided in the output, then save the current list of node processes:

   ```bash
   pm2 save
   ```

### Testing the App Tier

1. To test the app's health check endpoint, use the following command in your SSM terminal:

   ```bash
   curl http://localhost:4000/health
   ```

   You should receive:

   ```bash
   "This is the health check"
   ```

2. Test the database connection by hitting the following endpoint:

   ```bash
   curl http://localhost:4000/transaction
   ```

   You should see the response containing the sample data:

   ```json
   {
     "result": [
       { "id": 1, "amount": 400, "description": "groceries" },
       { "id": 2, "amount": 100, "description": "class" },
       { "id": 3, "amount": 200, "description": "other groceries" },
       { "id": 4, "amount": 10, "description": "brownies" }
     ]
   }
   ```

   If both tests succeed, your network, security, database, and app configurations are correct.

   <b>Congratulations! Your app layer is fully configured and operational.</b>

## Internal Load Balancing and Auto Scaling - Part 4

In this section, we'll create an Amazon Machine Image (AMI) of the app tier instance, then set up auto-scaling with a load balancer for high availability.

### Creating the App Tier AMI

1. Navigate to Instances on the left side of the EC2 dashboard. Select the app tier instance and under Actions, choose Image and Templates > Create Image.

   ![](/demos/CreateAMI1.png)

2. Name the image and provide a description, then click Create Image. Monitor the image creation status under AMIs on the left navigation panel.

   ![](/demos/CreateAMI2.png)

### Setting Up the Target Group

1. While the AMI is being created, navigate to Target Groups under Load Balancing in the EC2 dashboard. Click Create Target Group.

2. The target group will be used with the load balancer to balance traffic across the private app tier instances. Choose Instances as the target type, and set the protocol to HTTP and port to 4000 (where our Node.js app is running). Select the VPC and change the health check path to `/health`. Click Next.

3. Skip registering targets for now and create the target group.

### Deploying the Internal Load Balancer

1. Under Load Balancers in the EC2 dashboard, click Create Load Balancer and select Application Load Balancer.

2. Name the load balancer and set it to internal (non-public-facing) to route traffic from the web tier to the app tier.

   ![](/demos/LBConfig1.png)

   Configure the network settings for the VPC and private subnets.

   ![](/demos/LBConfig2.png)

   Assign the security group created for the internal ALB. Set the load balancer to listen for HTTP traffic on port 80 and forward it to the target group you just created. Click Create Load Balancer.

   ![](/demos/LBConfig3.png)

### Creating a Launch Template

1. Before configuring auto-scaling, create a Launch Template using the AMI created earlier. Navigate to Launch Template under Instances in the EC2 dashboard and click Create Launch Template.

2. Name the template, and under Application and OS Images, include the app tier AMI you created.

   ![](/demos/LaunchTemplateConfig1.png)

   Select `t2.micro` as the instance type. Leave Key pair and Network Settings blank, as we’ll set the network in the auto-scaling group.

   ![](/demos/LaunchTemplateConfig2.png)

   Assign the correct security group for the app tier, and under Advanced details, use the same IAM instance profile as before.

   ![](/demos/LaunchTemplateConfig3.png)
   ![](/demos/LaunchTemplateConfig4.png)

### Configuring Auto Scaling

1. Navigate to Auto Scaling Groups under Auto Scaling in the EC2 dashboard and click Create Auto Scaling Group.

2. Name the group, select the Launch Template created earlier, and proceed.

   ![](/demos/ConfigureASG1.png)

3. In the instance launch options, select your VPC and the private subnets for the app tier, then continue.

   ![](/demos/ConfigureASG2.png)

4. Attach the Auto Scaling Group to the internal load balancer by selecting the existing target group from the dropdown. Proceed to the next step.

   ![](/demos/ConfigureASG3.png)

5. Configure the group size and scaling policies with a desired, minimum, and maximum capacity of 2 instances. Review the settings and create the Auto Scaling Group.

   Your internal load balancer and auto-scaling group should now be correctly configured, with the auto-scaling group launching two new app tier instances. You can test the configuration

by manually terminating one instance and observing if a new instance is automatically launched.

<b>Note:</b> The original app tier instance used to create the AMI is excluded from the ASG, so you may see three instances in the EC2 dashboard. You can delete the original instance, but keeping it for troubleshooting is recommended.

## Deploying the Web Tier Instance - Part 5

Next, we’ll deploy an EC2 instance for the web tier and configure the NGINX web server and React.js website.

### Updating the Config File

Before creating the web instances, update the NGINX configuration file. Open the `application-code/nginx.conf` file from the repository and replace `[INTERNAL-LOADBALANCER-DNS]` on line 58 with your internal load balancer’s DNS entry. You can find this DNS entry in the internal load balancer’s details page.

![ReplaceCode](/demos/ReplaceCode.png)

Upload this file and the `application-code/web-tier` folder to the S3 bucket created for this project.

### Deploying the Web Instance

1. Follow the instance creation steps from **Part 3: App Tier Instance Deployment**, but this time use one of the **public subnets**. Ensure the correct network settings, security group, and IAM role are selected, and **auto-assign a public IP** during configuration.

   ![](/demos/WebInstanceCreate1.png)

   ![](/demos/WebInstanceCreate2.png)

2. Proceed without a key pair for this instance.

### Connecting to the Web Instance

1. Connect to the web instance using the same steps as for the app instance, and switch to the `ec2-user`. Test internet connectivity by pinging Google DNS:

   ```bash
   sudo -su ec2-user
   ping 8.8.8.8
   ```

   <b>Note:</b> If you don’t see a transfer of packets, verify the route tables attached to the subnet where your instance is deployed.

### Configuring the Web Instance

1. Install NVM and Node.js:

   ```bash
   curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.38.0/install.sh | bash
   source ~/.bashrc
   nvm install 16
   nvm use 16
   ```

2. Download the web tier code from the S3 bucket:

   ```bash
   cd ~/
   aws s3 cp s3://BUCKET_NAME/web-tier/ web-tier --recursive
   ```

3. Navigate to the web-layer folder, install dependencies, and create the build folder for the React app:

   ```bash
   cd ~/web-tier
   npm install
   npm run build
   ```

4. Install NGINX to serve the front-end application:

   ```bash
   sudo amazon-linux-extras install nginx1 -y
   ```

5. Replace the default NGINX configuration file with the one uploaded to S3:

   ```bash
   cd /etc/nginx
   sudo rm nginx.conf
   sudo aws s3 cp s3://BUCKET_NAME/nginx.conf .
   ```

   Restart NGINX:

   ```bash
   sudo service nginx restart
   ```

   Ensure NGINX can access the necessary files:

   ```bash
   chmod -R 755 /home/ec2-user
   ```

   Enable NGINX to start on boot:

   ```bash
   sudo chkconfig nginx on
   ```

6. Access your website by entering the public IP of the web tier instance in a browser. If the database connection is working, you will be able to interact with the site’s data.

   ![WebPage1](/demos/WebPage1.png)
   ![WebPage2](/demos/WebPage2.png)

## External Load Balancer and Auto Scaling - Part 6

In this final section, we’ll create an AMI of the web tier instance, set up auto-scaling, and deploy an external load balancer for high availability.

### Creating the Web Tier AMI

1. Navigate to Instances on the left side of the EC2 dashboard. Select the web tier instance and under **Actions** choose **Image and Templates > Create Image**.

2. Name the image and provide a description, then click **Create Image**. Monitor the creation status under **AMIs** on the left navigation panel.

### Setting Up the Target Group

1. While the AMI is being created, navigate to **Target Groups** under **Load Balancing** in the EC2 dashboard. Click **Create Target Group**.

2. This target group will balance traffic across the public web tier instances. Choose **Instances** as the target type, set the protocol to HTTP, and the port to 80 (where NGINX is listening). Select the VPC and set the health check path to `/health`. Click **Next**.

3. Skip registering targets for now and create the target group.

### Deploying the Internet-Facing Load Balancer

1. Under **Load Balancers** in the EC2 dashboard, click **Create Load Balancer** and select **Application Load Balancer**.

2. Name the load balancer and set it to **internet-facing** to route public traffic to the web tier.

   Configure the network settings for the VPC and **public subnets**.

   Assign the security group created for this ALB. Set the load balancer to listen for HTTP traffic on port 80 and forward it to the target group you just created. Click **Create Load Balancer**.

### Creating a Launch Template

1. Before configuring auto-scaling, create a Launch Template using the AMI created earlier. Navigate to **Launch Template** under **Instances** in the EC2 dashboard and click **Create Launch Template**.

2. Name the template, and under **Application and OS Images**, include the web tier AMI you created.

   ![](</demos/LaunchTemplateConfig1%20(1).png>)

   Select `t2.micro` as the instance type. Leave **Key pair** and **Network Settings** blank, as we’ll set the network in the auto-scaling group.

   Assign the correct security group for the web tier, and under **Advanced details**, use the same IAM instance profile as before.

### Configuring Auto Scaling

1. Navigate to **Auto Scaling Groups** under **Auto Scaling** in the EC2 dashboard and click **Create Auto Scaling Group**.

2. Name the group, select the Launch Template created earlier, and proceed.

3. In the instance launch options, select your VPC and the public subnets for the web tier, then continue.

4. Attach the Auto Scaling Group to the internet-facing load balancer by selecting the existing web tier target group from the dropdown. Proceed to the next step.

5. Configure the group size and scaling policies with a desired, minimum, and maximum capacity of 2 instances. Review the settings and create the Auto Scaling Group.

   Your external load balancer and auto-scaling group should now be correctly configured, with the auto-scaling group launching two new web tier instances. You can test the configuration by manually terminating one instance and observing if a new instance is automatically launched. Test the overall architecture by navigating to the external load balancer's DNS name in your browser.

   <b>Note:</b> The original web tier instance used to create the AMI is excluded from the ASG, so you may see three instances in the EC2 dashboard. You can delete the original instance, but keeping it for troubleshooting is recommended.

   ![FinalLBDNS](/demos/FinalLBDNS.png)

### **Congratulations! You've Successfully Implemented a Three-Tier Web Architecture!** <br>

## Clean-Up Instructions

To avoid ongoing charges, follow these steps to delete the resources in your AWS account in the specified order, starting with the services deployed inside the VPC, and then the VPC components themselves:

1. **Auto Scaling Groups**

   Delete the Auto Scaling Groups first, as they will continue to deploy EC2 instances if left active. In the EC2 console dashboard, select your app and web tier Auto Scaling Groups and delete them.

2. **Application Load Balancers**

   For each load balancer, delete the listeners first, then delete the load balancers by selecting them and choosing delete under the Actions dropdown.

3. **Target Groups**

   Once the load balancers are deleted, you can safely delete the associated target groups.

4. **Launch Templates**

   Navigate to Launch Templates in the EC2 dashboard, select the templates, and delete them. You will need to type **Delete** to confirm.

5. **AMIs**

   In the EC2 dashboard, navigate to AMIs under Images and deregister both of the AMIs created.

6. **Remaining EC2 Instances**

   If you haven't already deleted the instances used to create the AMIs, do so now by selecting them under Instances and choosing Terminate Instance.

7. **Aurora Database**

   To delete the database, navigate to the RDS service dashboard. Ensure that deletion protection is off by modifying the regional cluster settings. Then, delete the reader and writer nodes, ensuring you uncheck the snapshot option for the last instance.

8. **NAT Gateways**

   In the VPC dashboard, select and delete both NAT Gateways.

9. **Elastic IPs**

   Release any Elastic IPs by navigating to Elastic IPs under Virtual Private Cloud in the VPC dashboard.

10. **Route Tables**

    Remove all subnet associations for the route tables you created. Then, delete the route tables.

11. **Internet Gateway**

    Detach the Internet Gateway from the VPC, then delete it.

12. **VPC**

    Finally, delete the VPC, which will also remove the associated subnets and

security groups.

## License

This content is licensed under the MIT-0 License. See the LICENSE file for details.

---

This revised content maintains the original technical instructions while ensuring it is plagiarism-free.
