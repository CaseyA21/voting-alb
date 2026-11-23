# Deploy manually

This document describes the steps to set up the DynamoDB table, security groups, EC2 launch template, Elastic Load Balancer, and Auto-Scaling Group. 

## One-time setup

The file `deploy/userdata.sh` is used when an instance boots, and you must change one line before you deploy.

* Open `deploy/userdata.sh` in Cursor or `nano`.
* Near the top of the file you will find the line:

  ```
  REPO_URL="https://github.com/YOUR_GITHUB_USERNAME/voting-alb.git"
  ```

* Change `YOUR_GITHUB_USERNAME` to your GitHub username.
* Commit this change to the git repo, and push it back to your GitHub account.

  ```
  git add deploy/userdata.sh
  git commit -m "set github account"
  git push origin main
  ```


If the `git push` command fails, check the url of `origin` and make sure it points at your fork of the repo.

```
git remote -v
```


## Create the Polls table

The app calls DynamoDB in `us-east-1`. A table created in another region is a different table.

In the DynamoDB console, choose **Create table**.

* Table name: `Polls`
* Partition key: `pollId`, type String
* Table settings: **Default settings**


Create the table and wait until its status is Active. Gunicorn describes `Polls` when it starts, and it keeps restarting until that call succeeds.


## Create the security groups

The load balancer and the instances each use their own security group. Create `voting-balancer-sg` first because it is used by the `voting-instnace-sg`.


### `voting-balancer-sg`

These steps create a security group that allows HTTP from anywhere to the load balancer.

* In the EC2 console, open **Security Groups**.
* Click **Create security group**.
* For **Security group name** enter `voting-balancer-sg`.
* For **Description** enter, "SG for the load balancer allowing HTTP."
* Leave **VPC** as it is. It is the default VPC.
* Under **Inbound rules**, click **Add rule** and then set:
  * **Type**: "HTTP"
  * **Source type**: "Anywhere-IPv4"
* At the bottom of the page, click **Create security group**.


### `voting-instance-sg`

These steps create a security group that allows SSH from anywhere, and HTTP only from the load balancer.

* In the EC2 console, open **Security Groups**.
* Click **Create security group**.
* For **Security group name** enter `voting-instance-sg`.
* For **Description** enter, "SG for Voting instances allowing SSH and HTTP from the load balancer."
* Leave **VPC** as it is. It is the default VPC.
* Under **Inbound rules**, click **Add rule** and then set:
  * **Type**: "SSH"
  * **Source type**: "Anywhere-IPv4"
* Click **Add rule** again and then set:
  * **Type**: "HTTP"
  * **Source type**: "Custom"
  * Click the search icon next to **Source** and select the security group `voting-balancer-sg`.
* At the bottom of the page, click **Create security group**.


## Create the launch template

The launch template describes how each EC2 instance will be created.

In the EC2 console, open **Launch templates** and choose **Create launch template**.


* **Launch template name**: `voting-launch-template`
* **Application and OS Images**: Select "Quick Start" and then "Amazon Linux"
* **Instance type**: `t3.micro`
* **Key pair**: `vockey`
* **Network Settings**:
  * **Subnet**: "Don't include in launch template" (ELB uses two subnets in different Availability Zones).
  * **Security groups**: `voting-instance-sg`
* **Advanced details**:
  * **IAM instance profile**: `LabInstanceProfile`
  * **User data**: contents of `deploy/userdata.sh`



## Create the target group

The Elastic Load Balancer will send requests to instances in the target group, and Auto Scaling will add and remove instances to this group.

In the EC2 console, under **Load Balancing** in the left menu, open **Target groups** and choose **Create target group**.

* **Target type**: Instances. This is the default.
* **Target group name**: `voting-target-group`
* **Protocol**: HTTP, port 80. These are already set.
* **Health check path**: `/health`
* **Register targets** (2nd step): Do not add instances.  Auto Scaling will do this.


## Create the load balancer

In the EC2 console, under **Load Balancing** in the left menu, open **Load balancers**, and choose **Create load balancer**.

* **Type**: Application Load Balancer
* **Load balancer name**: `voting-load-balancer`
* **Scheme**: Internet-facing. This is the default.
* **IP address type**: IPv4. This is the default.
* **VPC**: Leave it as it is. It is the default VPC.
* **Network mapping**: Select Availability Zones `us-east-1a` and `us-east-1b`. In each zone, select its subnet.
* **Security groups**: Remove the default security group. Select `voting-balancer-sg`.
* **Listener**: HTTP, port 80. These are already set. Forward to `voting-target-group`.


## Create the Auto Scaling group

In the EC2 console, under **Auto Scaling** in the left menu, open **Auto Scaling groups** and choose **Create Auto Scaling group**.

* **Auto Scaling group name**: `voting-auto-scaling`
* **Launch template**: `voting-launch-template`
* **VPC**: Leave it as it is. It is the default VPC.
* **Availability Zones and subnets**: `us-east-1a` and `us-east-1b`
* **Load balancing**: Attach to an existing load balancer. Choose from your load balancer target groups, and select `voting-target-group`.
* **Health checks**: Turn on **Elastic Load Balancing** health checks so the group replaces an instance that fails `/health`. The grace period is 300 seconds. This is already set.
* **Group size**: Desired capacity 1 and minimum capacity 1 are already set. Set **Max desired capacity** to 3.
* **Scaling policies**: Choose **Target tracking scaling policy**. **Metric type** Average CPU utilization and **Target value** 50 are already set.


## Open the site

It will take Auto Scaling a few minutes to create the first instance and for it to become health.  When it is, open the load balancer’s DNS name in a browser.


## Delete the deployment

The order to delete resources matters because of dependencies.  Use the following order, and wait for each to fully delete before proceeding:

1. The Auto Scaling group `voting-auto-scaling`. Deleting the group terminates the instances it launched.
2. The load balancer `voting-load-balancer`.
3. The target group `voting-target-group`.
4. The launch template `voting-launch-template`.
5. The security groups `voting-instance-sg` and `voting-balancer-sg`.
6. The `Polls` table.

