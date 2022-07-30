---
title: Cloud Resume Challenge
date: 2022-01-10 12:00:00 -0400
image: 
  path: https://raw.githubusercontent.com/s4yhii/s4yhii.github.io/master/assets/images/htb/AWS-Blogging/banner.jpeg
  height: 1100
  width: 500
categories: [AWS, Blogging]
tags: [s3, challenge, cloud, aws, resume]
---

# Cloud Resume Challenge

## Setup AWS

Create your aws account 

Setup MFA for your roor account

Create an IAM user

Assign permission (Principle of Least privilege)

Setup Vault (https://github.com/99designs/aws-vault)

aws-vault add myuser ( ex: aws-vault add dev)

aws-vault exex myuser — aws s3 ls

## Setup S3

What is s3: file service useful for storing files usually for host a website

What is AWS SAM: server less application model

we will create an AWS Lambda (we ignore this for now)

Setups sam cli

sam init

sam build

Add IAM permissions all in full access(Cloud formation, IAM, Lambda, API Gateway)

Deploy Sam

aws-vault exec myuser —no-session — sam deploy --guided

before deploy add this resource to your yaml template

```yaml
MyWebsite:
Type: AWS::S3::Bucket
Properties:
BucketName: my-website
```

if deploys fails, delete your stack and deploy again

`aws cloudformation delete-stack --stack-name <<stack-name>>`

only the first time with the para  -- guided

Add S3

aws-vault exex myuser — aws s3 ls

## Push your website to S3

use this unified command

```yaml
sam build && aws-vault exec my-user --no-session -- sam deploy
```

to setup your s3 bucket as websitte edit your yaml file with this 

```yaml
MyWebsite:
    Type: AWS::S3::Bucket
    Properties:
      AccessControl: PublicRead
      WebsiteConfiguration:
        IndexDocument: index.html
      BucketName: my-fantastic-website

ONLY THE FIRST PART

 BucketPolicy:
    Type: AWS::S3::BucketPolicy
    Properties:
      PolicyDocument:
        Id: MyPolicy
        Version: 2012-10-17
        Statement:
          - Sid: PublicReadForGetBucketObjects
            Effect: Allow
            Principal: "*"
            Action: "s3:GetObject"
            Resource: !Join
              - ""
              - - "arn:aws:s3:::"
                - !Ref MyWebsite
                - /*
      Bucket: !Ref MyWebsite
THIS PART AFTER UPLOAD OUR HTML FILE TO OUR BUCKET
```

Then update your makefile file

```yaml
.PHONY: build

build:
	sam build

deploy-infra:
	sam build && aws-vault exec my-user --no-session -- sam deploy

deploy-site:
	aws-vault exec my-user --no-session -- aws s3 sync ./resume-site s3://my-fantastic-website
```

then upload your index.html 

```bash
make deploy-site
```

add some css for fancy view

```css
<head>
    <style>
        p {
            color: red;
        }
    </style>
</head>
```