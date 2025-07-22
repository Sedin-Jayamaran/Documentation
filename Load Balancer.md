# 🔧 Step-by-Step: Setup Load Balancer for ECS (Rails app)

```plaintext
Browser (Internet)
   ↓
Public ALB (in public subnet, has public IP)
   ↓
Target Group → EC2 instance (must be reachable from ALB)
   ↓
Docker container (your Rails app)
```
-------
## ✅ Step 1: Create an Application Load Balancer (ALB)
------
Go to EC2 > Load Balancers > Create Load Balancer

Select Application Load Balancer

Name: rails-alb

Scheme: Internet-facing

IP address type: IPv4

Listeners: HTTP (Port 80)

Availability Zones: select the same VPC & subnets as your EC2

Click Next
-----------
## ✅ Step 2: Create a Target Group
----------
Type: Instance (because you're using EC2 launch type, not Fargate)

Name: rails-target-group

Protocol: HTTP

Port: 80 (or 3000, match container port)

Target Type: instance

Health check:

    Protocol: HTTP
    Path: / or /health_check (whatever your Rails app supports)

Click Create
-----------
## ✅ Step 3: Register Your EC2 in the Target Group
--------
After creating the target group, go to Targets tab

Click Register targets

Choose your EC2 instance (the one running the Rails app)

Port: 80 (or 3000, match your container port)

Click Register
------------
## ✅ Step 4: Attach Target Group to the ALB Listener
-------------
Go back to Load Balancers > rails-alb

Select Listeners > View/Edit rules

Click on forward to...

Change to forward to your rails-target-group
-----------
## ✅ Step 5: Update ECS Rails App Service to Use the ALB
---------
Go to ECS > Clusters > YourCluster > Services > Rails App Service:

Click Update

Enable Load balancing

Select Application Load Balancer

Choose the ALB you created

Target group: rails-target-group

Port: 80 (or your app port)

Finish the update

----------------
``✅ Why does your EC2 instance need to be in a public subnet when using a Load Balancer (ALB) with ECS (EC2 launch type)?
Because you want external users (Internet) to reach your Rails app, and that only works if the EC2 instance hosting your container is reachable from the Load Balancer — which is public``
---------
```
🚀 After Setup – Test It
Go to EC2 > Load Balancers > rails-alb

Copy the DNS name (e.g., rails-alb-123456.ap-south-1.elb.amazonaws.com)

Open in browser:
→ http://rails-alb-xxxxxxxxxx.elb.amazonaws.com 
```