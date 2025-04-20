Perfect! Here's your content organized **exactly** the way you wrote it—with file names clearly labeled, followed by the corresponding **AWS CLI command or CloudFormation YAML** contents.

---

# ☕ Cafe CloudFormation Setup Guide

## #2. Set the ownership controls on the bucket

**Command:**
```bash
aws s3api put-bucket-ownership-controls \
  --bucket YOUR BUCKET NAME \
  --ownership-controls '{"Rules":[{"ObjectOwnership":"BucketOwnerPreferred"}]}'
```

---

## #3. Set the public access block settings on the bucket

**Command:**
```bash
aws s3api put-public-access-block \
  --bucket YOUR BUCKET NAME \
  --public-access-block-configuration '{
    "BlockPublicAcls": false,
    "IgnorePublicAcls": false,
    "BlockPublicPolicy": false,
    "RestrictPublicBuckets": false
  }'
```

---

## #4. Copy the website files to the bucket

**Command:**
```bash
aws s3 cp --recursive . s3://YOUR BUCKET NAME/ --acl public-read
```

---

## #5. `s3.yaml` updated file

**Content:**
```yaml
AWSTemplateFormatVersion: "2010-09-09"
Description: cafe S3 template

Resources:
  S3Bucket:
    Type: AWS::S3::Bucket
    DeletionPolicy: Retain
    Properties:
      WebsiteConfiguration:
        IndexDocument: index.html

Outputs:
  WebsiteURL:
    Description: URL for website hosted on S3
    Value: !GetAtt S3Bucket.WebsiteURL
```

---

## #6. `cafe-network.yaml` updated file

**Content:**
```yaml
AWSTemplateFormatVersion: 2010-09-09
Description: Network layer for the cafe

Resources:
  VPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: 10.0.0.0/16
      EnableDnsSupport: true
      EnableDnsHostnames: true
      Tags:
        - Key: Name
          Value: Cafe VPC

  IGW:
    Type: AWS::EC2::InternetGateway
    Properties:
      Tags:
        - Key: Name
          Value: Cafe IGW

  VPCtoIGWConnection:
    Type: AWS::EC2::VPCGatewayAttachment
    DependsOn:
      - IGW
      - VPC
    Properties:
      InternetGatewayId: !Ref IGW
      VpcId: !Ref VPC

  PublicRouteTable:
    Type: AWS::EC2::RouteTable
    DependsOn: VPC
    Properties:
      VpcId: !Ref VPC
      Tags:
        - Key: Name
          Value: Cafe Public Route Table

  PublicRoute:
    Type: AWS::EC2::Route
    DependsOn:
      - PublicRouteTable
      - VPCtoIGWConnection
    Properties:
      DestinationCidrBlock: 0.0.0.0/0
      GatewayId: !Ref IGW
      RouteTableId: !Ref PublicRouteTable

  PublicSubnet:
    Type: AWS::EC2::Subnet
    DependsOn: VPC
    Properties:
      VpcId: !Ref VPC
      MapPublicIpOnLaunch: true
      CidrBlock: 10.0.0.0/24
      AvailabilityZone: !Select
        - 0
        - !GetAZs
          Ref: AWS::Region
      Tags:
        - Key: Name
          Value: Cafe Public Subnet

  PublicRouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    DependsOn:
      - PublicRouteTable
      - PublicSubnet
    Properties:
      RouteTableId: !Ref PublicRouteTable
      SubnetId: !Ref PublicSubnet

Outputs:
  PublicSubnet:
    Description: The subnet ID to use for public web servers
    Value:
      Ref: PublicSubnet
    Export:
      Name:
        'Fn::Sub': '${AWS::StackName}-SubnetID'
  VpcId:
    Description: The VPC ID
    Value:
      Ref: VPC
    Export:
      Name:
        'Fn::Sub': '${AWS::StackName}-VpcID'
```

---

## #7. `cafe-app.yaml` updated file

**Content:**
```yaml
AWSTemplateFormatVersion: 2010-09-09
Description: Cafe application

Parameters:

  LatestAmiId:
    Type: 'AWS::SSM::Parameter::Value<AWS::EC2::Image::Id>'
    Default: '/aws/service/ami-amazon-linux-latest/amzn2-ami-hvm-x86_64-gp2'

  CafeNetworkParameter:
    Type: String
    Default: update-cafe-network

  InstanceTypeParameter:
    Type: String
    Description: Choose t2.micro, t2.small, t3.micro, or t3.small as EC2 instance type.
    Default: t2.small
    AllowedValues:
      - t2.micro
      - t2.small
      - t3.micro
      - t3.small

Mappings:

  RegionMap:
    us-east-1:
      "keypair": "vockey"
    us-west-2:
      "keypair": "cafe-oregon"

Resources:

  CafeSG:
    Type: 'AWS::EC2::SecurityGroup'
    Properties:
      GroupDescription: Enable SSH, HTTP access
      VpcId: !ImportValue
        'Fn::Sub': '${CafeNetworkParameter}-VpcID'
      Tags:
        - Key: Name
          Value: CafeSG
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: '80'
          ToPort: '80'
          CidrIp: 0.0.0.0/0
        - IpProtocol: tcp
          FromPort: '22'
          ToPort: '22'
          CidrIp: 0.0.0.0/0

  CafeInstance:
    Type: AWS::EC2::Instance
    Properties:
      ImageId: !Ref LatestAmiId
      InstanceType: !Ref InstanceTypeParameter
      KeyName: !FindInMap [RegionMap, !Ref "AWS::Region", keypair]
      IamInstanceProfile: CafeRole
      NetworkInterfaces:
        - DeviceIndex: '0'
          AssociatePublicIpAddress: 'true'
          SubnetId: !ImportValue
            'Fn::Sub': '${CafeNetworkParameter}-SubnetID'
          GroupSet:
            - !Ref CafeSG
      Tags:
        - Key: Name
          Value: Cafe Web Server
      UserData:
        Fn::Base64:
          !Sub |
            #!/bin/bash
            yum -y update
            yum install -y httpd mariadb-server wget
            amazon-linux-extras install -y lamp-mariadb10.2-php7.2 php7.2
            systemctl enable httpd
            systemctl start httpd
            systemctl enable mariadb
            systemctl start mariadb
            wget https://aws-tc-largeobjects.s3.us-west-2.amazonaws.com/CUR-TF-200-ACACAD-3-113230/15-lab-mod11-challenge-CFn/s3/cafe-app.sh
            chmod +x cafe-app.sh
            ./cafe-app.sh

Outputs:
  WebServerPublicIP:
    Value: !GetAtt 'CafeInstance.PublicIp'
```
