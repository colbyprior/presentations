# AWS Introduction

---

# Ec2 (Servers)

- Server charged based off instance size, per minute.
- Set up using local account with preloaded ssh key.
- Can have routine filesystem snapshots however most storage is ephemeral.
- Ec2 can have "User Data" which is code that is run on creation.

---

# Cloudformation (Infrastructure as code)

Cloudformation allows you to define your infrastructure in a yaml file for automated provisioning and updating.

It also allows you to pass input parameters, reference objects and substitute variables.

    !yaml
    AWSTemplateFormatVersion: "2010-09-09"
    Description: Mailman deploy cloudformation

    Parameters:
      Subnet:
        Type: List<AWS::EC2::Subnet::Id>
        Description: Subnet to deploy Ec2 in

    Resources:
      Ec2Instance: 
        Type: AWS::EC2::Instance
        Properties: 
          ImageId: ami-65a60007
          KeyName: my-ssh-key
          InstanceType: t2.micro
          SubnetId: !Ref Subnet

    Outputs:
      MyInstance:
        Description: My Ec2 instance
        Value: !Ref Ec2Instance
        Export:
          Name: !Sub "${AWS::StackName}-MyInstance"

---

# S3 (File storage)

- S3 is for flat file storage.
- It is not a NAS.
- S3 can have webhosting for static content.
- Versioning can be enabled on a S3 bucket.
- Server side encryption can be enabled on a S3 bucket.

Cloudformation S3 resource

    !yaml
    MyBucket:
      Type: AWS::S3::Bucket
      Properties: 
        BucketName: my_bucket

---

# Bringing Ec2 and S3 together

- The AWS CLI can perform automation in User Data scripts.
- `169.254.169.254` is a special IP used to get information about your instance.

Ec2 instance cloudformation to run ansible on localhost

    !yaml
    Type: AWS::EC2::Instance
    Properties:
      ImageId: ami-65a60007
      KeyName: my-ssh-key
      InstanceType: t2.micro
      SubnetId: !Ref Subnet
      UserData:
        "Fn::Base64":
          !Sub
           - |
            #!/bin/bash
            TMP_INSTALL_DIR='/etc/setup-files'
            CONFIG_BUCKET='${ConfigBucket}'
            apt update -y
            apt install awscli jq ansible -y
    
            REGION=$(curl -s http://169.254.169.254/latest/dynamic/instance-identity/document | jq .region -r)
            aws configure set default.region $REGION
            mkdir $TMP_INSTALL_DIR
            aws s3 cp s3://$CONFIG_BUCKET/ $TMP_INSTALL_DIR --recursive
            ansible-playbook $TMP_INSTALL_DIR/ansible.yml --connection=local

---

# IAM roles (Access control)

- IAM roles can be applied to users (via SAML) and Ec2 instances (and more).
- IAM policies can be attached to roles to dictate what access the role has.

Cloudformation for IAM role to access S3 bucket

    !yaml
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: "2012-10-17"
        Statement:
          - 
            Effect: "Allow"
            Principal:
              Service:
                - "ec2.amazonaws.com"
            Action:
              - "sts:AssumeRole"
      Policies:
        - PolicyName: "readMyBucket"
          PolicyDocument:
            Version: "2012-10-17"
            Statement:
              - 
                Effect: "Allow"
                Action:
                  - "s3:ListBucket"
                  - "s3:GetObject"
                Resource:
                  - !Join [ "", [ "arn:aws:s3:::", Ref: MyBucket, "/*" ] ]

---

# RDS (Databases)

- RDS has auto backup to S3 features.
- RDS can also perform auto upgrades for minor versions.
- RDS can create SQL slaves in different availability zones.


RDS cloudformation

    !yaml
    Type: AWS::RDS::DBInstance
    Properties:
      DBName: mydb
      DBSubnetGroupName: !Ref Subnet
      AllocatedStorage: 5
      AutoMinorVersionUpgrade: True
      DBInstanceClass: db.t2.micro
      Engine: MariaDB
      MasterUsername: db_user
      MasterUserPassword: !Ref DBPass
      PubliclyAccessible: False
      DBInstanceIdentifier: !Ref AWS::StackName
      BackupRetentionPeriod: 7
      MultiAZ: True

---

# Security groups (Firewall)

Cloudformation for ssh access to a Ec2 and database access

    !yaml
    Ec2Group:
     Type: AWS::EC2::SecurityGroup
     Properties:
       GroupDescription: This is the security group management access
       VpcId: !Ref VpcId
       SecurityGroupIngress:
         - 
          IpProtocol: "tcp"
          FromPort: "22"
          ToPort: "22"
          CidrIp: !Ref ManagementRange

    RDSGroup:
      Type: AWS::EC2::SecurityGroup
      DependsOn:
       - Ec2Group
      Properties:
        VpcId: !Ref VpcId
        SecurityGroupIngress:
          -
           IpProtocol: "tcp"
           FromPort: "3306"
           ToPort: "3306"
           SourceSecurityGroupId: !GetAtt Ec2Group.GroupId


---

# Horizontal Scaling

- Ec2 instances can deployed in autoscaling groups.
- Load balancers can attach Ec2 instances from autoscaling using health checks.

Example of a load balancer target group that an auto scaling group would reference.

    !yaml
    Type: "AWS::ElasticLoadBalancingV2::TargetGroup"
    Properties:
      HealthCheckIntervalSeconds: 30
      HealthCheckPath: /health-check.php
      HealthCheckPort: 80
      HealthCheckProtocol: HTTP
      HealthCheckTimeoutSeconds: 28
      HealthyThresholdCount: 2
      Matcher:
        HttpCode: 200
      Name: !Ref AWS::StackName
      Port: 80
      Protocol: HTTP
      TargetGroupAttributes:
        - Key: stickiness.type
          Value: lb_cookie
        - Key: stickiness.enabled
          Value: true
        - Key: stickiness.lb_cookie.duration_seconds
          Value: 3600
      UnhealthyThresholdCount: 2
      VpcId: !Ref VpcId

---

# VPC (Networking)

- A VPC can be split into tiers with different route tables.
- Each tier has three subnets in different physical locations.
- There can also be private VPN or direct connect links to VPC's.

![awsvpc](https://docs.aws.amazon.com/directconnect/latest/UserGuide/images/dx-gateway-diagram.png)


---

# ![awslamp](https://d1.awsstatic.com/partner-network/QuickStart/datasheets/drupal-architecture-on-the-aws-cloud.05b8a1305f655d046dc404c5b5079401c0030e1c.png)
