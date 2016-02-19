# AWS Stack

### Description
The goal of this stack is to obtain an ansible server and a webserver fully redundant with alerting and scalling
here is the code of my playbooks :
[Ansible-playbooks](https://github.com/s0nny78/ansible-playbooks)
Only host and tomcat are used

1 EC2 instance (AnsibleServer) with :
  a dedicated SecurityGroup for SSH only
  an IAM role with full access
  SDK of AWS preinstalled
  user data which will install ansible and download my playbooks

2 EC2 with ubuntu image (ClojureCollectorX) :
  a dedicated SecurityGroup for SSH, HTTP and HTTP:8080
  a special tag to deploy ansible

1 ELB with all ClojureCollectorX :
  listen and redirect to : 80 & 8080
  HealthCheck : 5*30sec -> Service Out; 3*30sec -> InService

1 Autoscaling group for ClojureCollectorX :
  same flavor as ClojureCollectorX
  can launch 2 instances if the CPU is over loaded (+70%) and will send an email
  
Some outputs to retreive information easily

  
### How to :
```sh
aws cloudformation create-stack --stack-name suiton --capabilities CAPABILITY_IAM --template-body stack5.1
```

### Requierements
You will need a keypair named ec2key to do that
and put it in your own bucket with private access

line 63 change tadjer for your own name of bucket:
```sh
"sudo aws s3 cp s3://tadjer/ec2key.pem /home/ec2-user/.ssh/ \n",
```

line 274 change the email
```sh
"Endpoint" : "nospam@gmail.com",
```

### About me
If you want to contact me @HouariTADJER

### Improve it
I am sure you can do better, do not hesitate to create a branch to improve it :
- Replay the playbook and/or Launch a new instance if the port 8080 of a ClojureCollectorX is down 
- Improve the host file for Ansible Server using template and not task
- exclude the region us-east-1e which does not support m1.small
- create a full VPC, IGW and Subnet to isolate it the stack
- give rights to Clojure-collector VM for S3 bucket to write logs
- find a better names for thos different objects
- let the user choose the flavor of machine, AZ, etc...
- reduce or avoid to use user-data (use Ansible or configure an AMI)
- reduce the IAM power of Ansible Server
- improve the way to give ec2key to ansible server
- Change the inbound of SG
- remove the public IP of ClojureCollectorX
- add a cloudfront
- add a route53 to ELB
- add cloudwatch alarm for other things than CPU
