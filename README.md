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
  HealthCheck : 5x30sec -> Service Out; 3x30sec -> InService

1 Autoscaling group for ClojureCollectorX :
  same flavor as ClojureCollectorX
  can launch 2 instances if the CPU is over loaded (+70%) and will send an email
  
Some outputs to retreive information easily

  
### How to :

To launch the stack with IAM capabilities :
```sh
aws cloudformation create-stack --stack-name suiton --capabilities CAPABILITY_IAM --template-body stack5.2.json
```

After it is launched, you will have to connect to ansible-server and deploy the playbook tomcat8 (which takes around 5 min to complete)
```sh
ansible-playbook /etc/ansible/playbooks/tomcat/tomcat.yml
```

You can try the autoscaling with stress on the hosts :
```sh
apt-get install stress
```

and then :
```sh
stress -c 8
```
If you want to know more about it :
http://www.cyberciti.biz/faq/stress-test-linux-unix-server-with-stress-ng/


### Comments :
Create an 3 instances with :
```sh
AWS::EC2::Instance
```

Define bootstrapp : 
```sh
UserData
```
What it does :
Install and configure ansible
Install git and DL my playbooks
DL the ssh key and configure sshd to ansible
Deploy the playbook host to have the new list with client
Deploy the playbook tomcat to install it to clients (currently not working)


2 SG created, one for Ansible server (22) and the other for webapps (22 ssh, 80 http and 8080 default tomcat port)
```sh
AWS::EC2::SecurityGroup
SecurityGroupIngress -> CidrIp": "0.0.0.0/0 (open to the world, this is bad and I should feel bad)
```

When you create an EIP, you should associate it
```sh
"IPAddress1" : {
      "Type" : "AWS::EC2::EIP"
      },

"IPAssoc1" : {
      "Type" : "AWS::EC2::EIPAssociation",
        "Properties" : {
          "InstanceId" : { "Ref" : "ClojureCollector1" },
          "EIP" : { "Ref" : "IPAddress1" }
        }
      },
```

We do need an IAM new profile and role which will have a policy with full rights except IAM 

```sh
"PolicyDocument" : {
            "Statement" : [ {
                "Effect" : "Allow",
                "NotAction" : "iam:*",
                "Resource" : "*"
            }]
          },
```

Creation of an ELB, which will forward the port 80 and 8080 
```sh
"Listeners": [
          {
          "LoadBalancerPort": "8080",
          "InstancePort": "8080",
          "Protocol": "HTTP"
          },
```

and check the health of 8080 only, can be improved to put clojure modul instead of /
```sh
 "HealthCheck": {
        "Target": "HTTP:8080/",
        "HealthyThreshold": "3",
        "UnhealthyThreshold": "5",
        "Interval": "30",
        "Timeout": "5"
        },
```
3x30 secondes to be checked as healthy
5x30 secondes to be checked as unhealthy 
5 secondes of timeout to receive and answer for each Threshold


Define the metadata of the new instance
```sh
"LaunchConfig" : {
      "Type" : "AWS::AutoScaling::LaunchConfiguration",
        "Properties" : {
        "KeyName" : "ec2key",
        "ImageId" : "ami-b2e3c6d8",
        "SecurityGroups" : [ { "Ref" : "InstanceSecurityGroup" } ],
        "InstanceType" : "m1.small"
        }
      },
```

Define the autoscaling group, with limit for number of instances, where to put them (ELB) and to give them tag
````sh
AWS::AutoScaling::AutoScalingGroup
```

Define how the autoscaling group will increase :
```sh
 "ScaleUpPolicy" : {
      "Type" : "AWS::AutoScaling::ScalingPolicy",
        "Properties" : {
        "AdjustmentType" : "ChangeInCapacity",
        "AutoScalingGroupName" : { "Ref" : "WebServerGroup" },
        "Cooldown" : "300",
        "ScalingAdjustment" : "1"
        }
  },
```
will increase only 1 and wait 5 minutes before adding another one if necesseray
      

Define a push notification by mail 

```sh
"MySNSTopic" : {
      "Type" : "AWS::SNS::Topic",
        "Properties" : {
          "Subscription" : [ {
            "Endpoint" : "nospam@gmail.com",
            "Protocol" : "email"
          } ]
        }
      },
```

And the most important point define the alaerm and the actions taken :
```sh
AWS::CloudWatch::Alarm
"AlarmActions" : [ { "Ref" : "MySNSTopic" }, { "Ref": "ScaleUpPolicy" } ],
"Threshold" : "70",
```
CPU > 70%


And finally objects/variable to retreive from the stack console :
```sh
"Outputs" : {
"PrivateIP" : {
       "Description" : "Private IP address of the newly created EC2 instance",
       "Value" : { "Fn::GetAtt" : [ "ClojureCollector1", "PrivateIp" ] }
     },
}
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
