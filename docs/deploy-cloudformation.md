# Deploy with CloudFormation (CLI)

This document explains how to launch the voting app with **AWS CloudFormation** and the **AWS CLI**. One stack creates the Polls table, the security groups, the launch template, the load balancer, and the Auto Scaling group. Deleting the stack deletes them.


## Prerequisites

You need the **AWS CLI** installed and configured with current AWS Academy Learner Lab credentials in the `[default]` profile. The commands below use region `us-east-1`.


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


The stack gives these resources fixed names: `Polls`, `voting-balancer-sg`, `voting-instance-sg`, `voting-launch-template`, `voting-target-group`, `voting-load-balancer`, and `voting-auto-scaling`. If any of them already exist, delete them before you create the stack. The order is in [Deploy manually](deploy-manual.md).


## Create the stack

`deploy/cloudformation.yaml` creates:

* A DynamoDB table named `Polls` with partition key `pollId`
* Security group `voting-balancer-sg`, allowing HTTP from anywhere
* Security group `voting-instance-sg`, allowing SSH from anywhere and HTTP from `voting-balancer-sg`
* Launch template `voting-launch-template` (`t3.micro`, key `vockey`, `LabInstanceProfile`)
* Target group `voting-target-group`, health check path `/health`
* Application load balancer `voting-load-balancer` in subnets `us-east-1a` and `us-east-1b`, forwarding HTTP port 80 to the target group
* Auto Scaling group `voting-auto-scaling` (minimum 1, desired 1, maximum 3) with a target tracking policy on average CPU utilization at 50


The group launches instances after the table is active. From the root of your clone, look up the default VPC and the default subnet in each Availability Zone, then create the stack. Replace `YOUR_GITHUB_USERNAME` with your GitHub username.

```bash
VPC_ID=$(aws ec2 describe-vpcs \
  --region us-east-1 \
  --filters Name=is-default,Values=true \
  --query 'Vpcs[0].VpcId' \
  --output text)

SUBNET_A=$(aws ec2 describe-subnets \
  --region us-east-1 \
  --filters Name=default-for-az,Values=true Name=availability-zone,Values=us-east-1a \
  --query 'Subnets[0].SubnetId' \
  --output text)

SUBNET_B=$(aws ec2 describe-subnets \
  --region us-east-1 \
  --filters Name=default-for-az,Values=true Name=availability-zone,Values=us-east-1b \
  --query 'Subnets[0].SubnetId' \
  --output text)

aws cloudformation create-stack \
  --region us-east-1 \
  --stack-name voting-alb \
  --template-body file://deploy/cloudformation.yaml \
  --parameters \
    ParameterKey=GitHubUsername,ParameterValue=YOUR_GITHUB_USERNAME \
    ParameterKey=VpcId,ParameterValue="$VPC_ID" \
    ParameterKey=SubnetIdA,ParameterValue="$SUBNET_A" \
    ParameterKey=SubnetIdB,ParameterValue="$SUBNET_B"
```


Wait until the stack finishes creating:

```bash
aws cloudformation wait stack-create-complete \
  --region us-east-1 \
  --stack-name voting-alb
```


User data installs the app and starts Gunicorn. Gunicorn calls Amazon DynamoDB with `LabInstanceProfile`. It does not use the `local` profile.


## After the stack is complete

Read the load balancer DNS name from the stack output:

```bash
aws cloudformation describe-stacks \
  --region us-east-1 \
  --stack-name voting-alb \
  --query 'Stacks[0].Outputs' \
  --output table
```


**CREATE_COMPLETE does not mean the app is ready.** User data still installs packages, clones the repo, and starts Gunicorn. Wait a few minutes, then:

```bash
curl -s http://LOAD_BALANCER_DNS_NAME/health
```


A 200 means Gunicorn can describe the `Polls` table. Open `http://` and that DNS name in a browser. SSH to an instance's public IPv4 address and check `/var/log/cloud-init-output.log` if something failed. The instance is listed on the Auto Scaling group's **Instance management** tab.


## Delete the stack

When you are done:

```bash
aws cloudformation delete-stack \
  --region us-east-1 \
  --stack-name voting-alb

aws cloudformation wait stack-delete-complete \
  --region us-east-1 \
  --stack-name voting-alb
```


Deleting this stack removes the Auto Scaling group and its instances, the load balancer, the target group, the launch template, both security groups, and the `Polls` table, including the polls stored in it.
