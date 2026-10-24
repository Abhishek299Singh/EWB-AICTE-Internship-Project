# AWS-VPC-Peering-and-EC2-Instance-Connectivity

# Creating VPC and Subnet

Login to the AWS console and navigate to the VPC section.<br>
Click on "Create VPC"<br><br>

![1](https://github.com/harshartz/AWS-VPC-Peering-and-EC2-Instance-Connectivity/assets/130890384/3d9ec5cd-f4aa-4188-a2e5-21c7f19bf6a3)
![2](https://github.com/harshartz/AWS-VPC-Peering-and-EC2-Instance-Connectivity/assets/130890384/4b5b7e42-44af-4bd0-86ad-44ce06797423)

provide a name tag for your VPC (e.g., "vpc-A").<br>
Define the CIDR range for your VPC (e.g., "10.0.0.0/16").<br>
Leave the other settings as default and click on "Create VPC".<br>

![3](https://github.com/harshartz/AWS-VPC-Peering-and-EC2-Instance-Connectivity/assets/130890384/7ade994a-42d5-4a88-9f55-7234976115c9)


On the left side of the VPC dashboard, select "Subnets."<br>
Click on "Create Subnet."<br>

![5](https://github.com/harshartz/AWS-VPC-Peering-and-EC2-Instance-Connectivity/assets/130890384/4846b30c-d8d2-406a-a0c3-37028296614e)

Select the VPC you just created.<br>
Set the CIDR range for the subnet, e.g., "10.0.1.0/24."<br>
Create the subnet.<br>

![6](https://github.com/harshartz/AWS-VPC-Peering-and-EC2-Instance-Connectivity/assets/130890384/007646cf-0275-4cb6-9370-e48f8eb94288)
![7](https://github.com/harshartz/AWS-VPC-Peering-and-EC2-Instance-Connectivity/assets/130890384/b61927de-52a3-4e32-b7bd-c3f64df10a82)

# Configuring Internet Gateway

Go back to the dashboard and select "Internet Gateway".<br>
Create an internet gateway using the VPC you just created.<br>

![8](https://github.com/harshartz/AWS-VPC-Peering-and-EC2-Instance-Connectivity/assets/130890384/5fffdcea-fb48-434f-9f5d-804490c013ba)
![9](https://github.com/harshartz/AWS-VPC-Peering-and-EC2-Instance-Connectivity/assets/130890384/7d83cf81-f79f-49a9-a32e-593a3913a7c8)

After creating the internet gateway, click on "Actions" at the top right side and select "Attach to VPC".<br>

![10](https://github.com/harshartz/AWS-VPC-Peering-and-EC2-Instance-Connectivity/assets/130890384/6f2fb7a9-0fd9-4870-8f03-850a9433077a)
![11](https://github.com/harshartz/AWS-VPC-Peering-and-EC2-Instance-Connectivity/assets/130890384/4af84412-f516-4746-ab50-18d58e9ba3a6)

# Setting Up Route Table

Navigate to the "Route Table" in the dashboard.<br>
To make it easier to understand, rename the already created route table in your VPC.<br>

![12](https://github.com/harshartz/AWS-VPC-Peering-and-EC2-Instance-Connectivity/assets/130890384/6cf1c6a5-d50b-4ea1-845a-2bc85f9753b2)
![13](https://github.com/harshartz/AWS-VPC-Peering-and-EC2-Instance-Connectivity/assets/130890384/0acc03f5-2f51-4201-8eb1-3123eeb2cb5e)

In the route table, click on "Edit routes".<br>
To allow your subnet to access the internet, add a new route to the subnet route table with the following settings:<br>

Destination: 0.0.0.0/0<br>
Target: The internet gateway that you just created<br><br>

![14](https://github.com/harshartz/AWS-VPC-Peering-and-EC2-Instance-Connectivity/assets/130890384/63066f54-19e3-44b6-a71f-5a6645d4503a)

Go to the "Subnet associations" tab in the route table.<br>

![15](https://github.com/harshartz/AWS-VPC-Peering-and-EC2-Instance-Connectivity/assets/130890384/89093d34-e4f0-44a4-a899-668d4d6e113d)

Click on "Edit subnet association" and select the subnet you created.<br>
Save the associations.<br>

![16](https://github.com/harshartz/AWS-VPC-Peering-and-EC2-Instance-Connectivity/assets/130890384/4d778a29-60c6-4e11-91e4-dfc65618c178)


# Creating Security Group

Scroll down on the dashboard and navigate to "Security Groups".<br>
Click on "Create security group" and provide a name for the security group.<br>

![17](https://github.com/harshartz/AWS-VPC-Peering-and-EC2-Instance-Connectivity/assets/130890384/493cd1ee-d5e0-4553-a03f-bbf39f362867)

Select your VPC.

![18](https://github.com/harshartz/AWS-VPC-Peering-and-EC2-Instance-Connectivity/assets/130890384/8f94e02e-8403-476d-a3a0-1627443ebbe6)

Click on "Edit inbound rules" and add a rule for "All ICMP IPv4" with the source set to "Anywhere - IPv4".<br>
Save the rules.<br>

![19](https://github.com/harshartz/AWS-VPC-Peering-and-EC2-Instance-Connectivity/assets/130890384/fbdaecee-1856-4cb2-a457-511e774f41f4)

# Launching EC2 Instance

Go to the EC2 section.<br>
Click on "Launch instance" and select a name tag for your instance.<br>

![21](https://github.com/harshartz/AWS-VPC-Peering-and-EC2-Instance-Connectivity/assets/130890384/a56c0839-b625-491a-ad4c-6efd263de26e)

Select an Amazon Machine Image (AMI) and Instance Type

![22](https://github.com/harshartz/AWS-VPC-Peering-and-EC2-Instance-Connectivity/assets/130890384/57831745-4a6e-4ee7-a2e5-3279d2f231c9)

Create a new key pair (e.g., "peering-A") or use an existing one.

![24](https://github.com/harshartz/AWS-VPC-Peering-and-EC2-Instance-Connectivity/assets/130890384/4c89ad59-6987-485c-b248-aa6f838b6c34)

Scroll down and edit the "Network Setting".<br>
Select your VPC and enable auto-assign public IP.<br>
Select the existing security group you created.<br>
Click on "Launch instance" and connect to the instance.<br>

![25](https://github.com/harshartz/AWS-VPC-Peering-and-EC2-Instance-Connectivity/assets/130890384/61a417a6-386f-469c-ac63-15f1cf143b34)


# Creating Second VPC and Subnet

Repeat the above steps to create another VPC called "vpc-B".<br>
Use CIDR range 172.16.0.0/16 for the VPC and 172.16.1.0/24 for the subnet.<br>

![26](https://github.com/harshartz/AWS-VPC-Peering-and-EC2-Instance-Connectivity/assets/130890384/291e98a5-43ab-4284-a80c-7fb0381853a1)

Launch an EC2 instance named "linux-B" in vpc-B.

![27](https://github.com/harshartz/AWS-VPC-Peering-and-EC2-Instance-Connectivity/assets/130890384/51bae2aa-1263-400f-9c81-33ca802025f3)

# Setting Up VPC Peering

Go to the VPC dashboard and navigate to "VPC Peering".<br>
Select "Create VPC Peering"<br>

![28](https://github.com/harshartz/AWS-VPC-Peering-and-EC2-Instance-Connectivity/assets/130890384/15397317-0f23-4109-b379-2b9534114a3f)

Give it a name (e.g., "peering-AB").<br>
Set "VPC-A" as the requester, "my account" as the accepter, and "VPC-B" as the select another VPC.<br>
Click on "Create Peering Connection".<br>

![29](https://github.com/harshartz/AWS-VPC-Peering-and-EC2-Instance-Connectivity/assets/130890384/0ab4dfb4-88d8-4100-9652-f178b4eaa47e)

In the "Actions" menu at the top right side, select "Accept Request" to accept the peering connection.

![30](https://github.com/harshartz/AWS-VPC-Peering-and-EC2-Instance-Connectivity/assets/130890384/d93e6df5-cd64-463d-8b2e-648391d70597)

# Configuring Route Tables for VPC Peering

Go to the VPC dashboard and navigate to the route tables.<br>

![31](https://github.com/harshartz/AWS-VPC-Peering-and-EC2-Instance-Connectivity/assets/130890384/6ade3ab6-b20a-4822-9d42-2cec05af669f)

Click on "Edit routes" for the route table of "vpc-A".<br>
Add a new route with the destination as the IP of "vpc-B" and the target as "VPC Peering".<br>

![32](https://github.com/harshartz/AWS-VPC-Peering-and-EC2-Instance-Connectivity/assets/130890384/009503ab-b6f4-4b27-a64f-5ba28c94fb5e)

Repeat the above step for the route table of "vpc-B", adding a rule with the destination as the IP of "vpc-A" and the target as "VPC Peering".

![33](https://github.com/harshartz/AWS-VPC-Peering-and-EC2-Instance-Connectivity/assets/130890384/482506fb-a053-4d48-8de4-e770f6c06afe)


# Connecting to EC2 Instance

To establish a connection between the EC2 instances, follow these steps:<br>

Connect to one of the EC2 instance<br>

![34](https://github.com/harshartz/AWS-VPC-Peering-and-EC2-Instance-Connectivity/assets/130890384/8c46ffea-b567-4ed6-94a8-d5479b21d701)

Switch to the root user:<br>

Run the command: <br>
```diff
-sudo -i
```

![36](https://github.com/harshartz/AWS-VPC-Peering-and-EC2-Instance-Connectivity/assets/130890384/022b53d9-cb3d-4068-b72f-c26d73078e73)

Create an empty file with a name of target Ec2's key pair file (e.g., "peering-B"):<br>

Run the command: <br>
```diff
-touch peering-B
```

Edit the file and paste the private key of the EC2 instance that you want to connect to:<br>

Run the command: <br>
```diff
-vi peering-B
```

![37](https://github.com/harshartz/AWS-VPC-Peering-and-EC2-Instance-Connectivity/assets/130890384/4b1c3576-6ac6-4a12-a402-ae3065c8167d)
![39](https://github.com/harshartz/AWS-VPC-Peering-and-EC2-Instance-Connectivity/assets/130890384/ecc075a8-df0b-41ea-8e9f-b98e3b066de9)

Modify the permissions of the file:<br>

Run the command: <br>
```diff
-chmod 400 peering-B
```

Use the SSH command to establish the connection to the other EC2 instance:<br>

Run the command: <br>
```diff
-ssh -i <<key-pair-file-name>> ec2-user@<<Target Linux EC2's Private IP>>
```
![40](https://github.com/harshartz/AWS-VPC-Peering-and-EC2-Instance-Connectivity/assets/130890384/e5a24844-c851-4cf8-b05d-0f686e4b23e0)

Select "yes" to confirm the connection

![41](https://github.com/harshartz/AWS-VPC-Peering-and-EC2-Instance-Connectivity/assets/130890384/896b9de1-0aa1-4eab-9008-30b728b84444)

![42](https://github.com/harshartz/AWS-VPC-Peering-and-EC2-Instance-Connectivity/assets/130890384/f81d441e-8d53-4c27-b868-2733bde11e18)

# You have successfully established a connection between the two EC2 instances.























