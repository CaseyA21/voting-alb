# Deploy manually

This document explains how to deploy the app from the AWS console. Stay in **US East (N. Virginia)** for every step. The DynamoDB table and the EC2 resources have to be in the same region.

Create the resources in the order below. The Auto Scaling group launches the instances. Creating the launch template does not start an instance.


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

In the EC2 console, open **Security groups** and create two groups in the default VPC. Leave the outbound rules as the console sets them. Create `voting-alb-balancer` first. The instance group’s HTTP rule uses that group as its source.

**voting-alb-balancer** allows the internet to reach the load balancer.

* Inbound rule: type HTTP, port 80, source `0.0.0.0/0`


**voting-alb-instances** allows the load balancer to reach the app, and allows SSH so you can inspect a running instance.

* Inbound rule: type HTTP, port 80, source `voting-alb-balancer`
* Inbound rule: type SSH, port 22, source `0.0.0.0/0`


## Create the launch template

`deploy/userdata.sh` does the following on every instance the group launches:

* Install Python and git
* Clone the repo
* Create the virtualenv
* Start Gunicorn


In the EC2 console, open **Launch templates** and choose **Create launch template**.

* Launch template name: `voting-alb`
* Application and OS Images: Amazon Linux 2023 (64-bit x86)
* Instance type: `t3.micro`
* Key pair: `vockey`
* Network settings: do not set a subnet. Choose the existing security group `voting-alb-instances`. Set **Auto-assign public IP** to Enable.
* Advanced details: IAM instance profile `LabInstanceProfile`
* Advanced details: paste the contents of `deploy/userdata.sh` into **User data**
* Resource tags: key `Name`, value `voting-alb`, applied to instances


Create the template, then continue with the target group. The Auto Scaling group starts the instances.


## Create the target group

The target group is the list of instances that receive requests. The Auto Scaling group adds and removes instances. You do not register an instance yourself.

In the EC2 console, open **Target groups** and choose **Create target group**.

* Target type: Instances
* Target group name: `voting-alb`
* Protocol: HTTP, port 80
* VPC: the default VPC
* Health check path: `/health`
* Healthy threshold: 2
* Success codes: `200`


On the **Register targets** step, do not add an instance. Create the target group.


## Create the load balancer

In the EC2 console, open **Load balancers**, choose **Create load balancer**, and choose **Application Load Balancer**.

* Name: `voting-alb`
* Scheme: Internet-facing
* IP address type: IPv4
* VPC: the default VPC
* Mappings: two subnets in different Availability Zones
* Security group: `voting-alb-balancer` (remove the default security group if the console selected it)
* Listener: protocol HTTP, port 80, forward to the target group `voting-alb`


Create the load balancer. Its DNS name is the address of the site. Copy that name. You will open it after an instance is healthy.


## Create the Auto Scaling group

In the EC2 console, open **Auto Scaling groups** and choose **Create Auto Scaling group**.

* Name: `voting-alb`
* Launch template: `voting-alb`
* VPC: the default VPC
* Subnets: the same two subnets you selected for the load balancer
* Load balancing: attach the existing target group `voting-alb`
* Health checks: turn on Elastic Load Balancing health checks
* Health check grace period: `300` seconds
* Desired capacity: `1`
* Minimum capacity: `1`
* Maximum capacity: `3`
* Scaling policy: target tracking on **Average CPU utilization**, target value `50`


Create the group. It launches one instance. User data takes a few minutes. On the target group’s **Targets** tab, the instance status becomes **healthy** after `/health` returns 200. Refresh that tab until it does.


## Open the site

Open `http://` and the load balancer’s DNS name in a browser. The same host serves the pages and the API. `/health` returns 200 when the instance can describe the `Polls` table.

```
curl -s http://LOAD_BALANCER_DNS_NAME/health
```


The public IPv4 address of an instance changes each time the group replaces it. Use the load balancer’s DNS name for the site. Use an instance’s public IPv4 address only for SSH.


## Debugging

In the EC2 instance list, find the instance with the Name tag `voting-alb`. SSH to its current public IPv4 address.

The Cloud-init process writes the userdata output to `/var/log/cloud-init-output.log`. If the app does not start, read that file.


## Other useful commands on the EC2 instance

- `systemctl status voting` — see the status of the Gunicorn process
- `curl -s http://localhost/health` — call `/health`. A 200 means the web server is running and it can describe the `Polls` table
- `sudo systemctl restart voting` — restart the web process after a config or code change
- `sudo journalctl -u voting -f` — follow the voting (gunicorn) logs


## Delete the deployment

Delete the resources in this order.

1. The Auto Scaling group `voting-alb`. Deleting the group terminates the instances it launched.
2. The load balancer `voting-alb`.
3. The target group `voting-alb`.
4. The launch template `voting-alb`.
5. The security groups `voting-alb-instances` and `voting-alb-balancer`.
6. The `Polls` table.


Wait until each delete finishes before you delete the next one. A security group stays in use until the load balancer and the instances are gone. Deleting the table deletes the polls stored in it.
