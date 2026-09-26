# 🌍 AWS Static Website Hosting with CloudFront, Route 53, ACM & Lambda Invalidation

### 📌 Introduction

This project demonstrates how to securely host a static website using AWS managed services with global content delivery, HTTPS security, and automated cache invalidation.

The website:

https://aniljadhav.co.in

is built using a private Amazon S3 bucket, delivered globally through CloudFront CDN, secured with SSL via AWS Certificate Manager, and automated with Lambda to handle cache invalidation whenever content is updated.

## 🏗 Architecture Diagram

![AWS Static Website Architecture](https://raw.githubusercontent.com/aniljadhavmca/Static-Website-Hosting-with-CloudFront-Route-53-ACM-Lambda-Invalidation/main/CloudFront,%20Route%2053,%20ACM%20%26%20Lambda%20Invalidation.png)


### 🌎 Amazon CloudFront (CDN)

CloudFront is a Content Delivery Network (CDN).
- Caches content at edge locations worldwide
- Reduces latency
- Handles HTTPS termination
- Protects the S3 bucket from direct access

Why CloudFront?
- Global performance improvement
- Built-in DDoS protection
- HTTPS support
- Edge caching for scalability

Without CloudFront:
- S3 would need to be public
- Performance would be slower globally
- No advanced caching control

---


### 🚀 Step-by-Step Setup Guide

## 1️⃣ Create S3 Bucket (Private)

- Name: aniljadhav.co.in
- Region: ap-south-1
- Block Public Access: ON
- Do NOT enable static website hosting
- Do NOT make bucket public


## 2️⃣ Upload Website Files

Upload `index.html` to the root of the bucket.

## 3️⃣ Create CloudFront Distribution (Detailed)

Go to: AWS Console → CloudFront → Create Distribution

### 🔹 Step 3.1: Origin Settings

- Origin domain:
  Select your S3 bucket  
  (⚠ NOT the S3 website endpoint)

- Origin access:
  Select → **Origin Access Control (OAC)**  
  If not created → Create new OAC  
  Signing behavior: **Sign requests (recommended)**

- Origin type:
  S3

Click Next.

### 🔹 Step 3.2: Default Cache Behavior

- Viewer protocol policy:
  **Redirect HTTP to HTTPS**

- Allowed HTTP methods:
  GET, HEAD

- Cache policy:
  Managed-CachingOptimized

- Compress objects automatically:
  Yes


### 🔹 Step 3.3: Distribution Settings

- Alternate Domain Name (CNAME):
  aniljadhav.co.in

- Custom SSL Certificate:
  Select ACM certificate (must be in us-east-1)

- Supported HTTP versions:
  HTTP/2 and HTTP/3

- Default Root Object:
  index.html

Click Create Distribution.


### 🔹 Step 3.4: Wait for Deployment

After creation:

Status will show:
"In Progress"

Wait until:
"Deployed"

This may take 5–10 minutes.

### 🔹 Step 3.5: Update S3 Bucket Policy (If Prompted)

CloudFront may prompt to update S3 bucket policy.

If not automatically updated, manually add OAC policy:

See Step 4 in README for policy JSON.

## 4️⃣ S3 Bucket Policy for OAC

Go to S3 → Permissions → Bucket Policy

Replace YOUR_ACCOUNT_ID and YOUR_DISTRIBUTION_ID before using:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowCloudFrontOACRead",
      "Effect": "Allow",
      "Principal": {
        "Service": "cloudfront.amazonaws.com"
      },
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::aniljadhav.co.in/*",
      "Condition": {
        "StringEquals": {
          "AWS:SourceArn": "arn:aws:cloudfront::YOUR_ACCOUNT_ID:distribution/YOUR_DISTRIBUTION_ID"
        }
      }
    }
  ]
}
```

## 5️⃣ Create ACM Certificate

- Switch region to: us-east-1
- Request Public Certificate
- Add domain: aniljadhav.co.in
- Validate using Route 53 DNS

⚠ CloudFront requires ACM certificate in us-east-1.


## 6️⃣ Configure Route 53
- Create Hosted Zone: aniljadhav.co.in
- Create A record Type: A, Alias: Yes, Target: CloudFront Distribution


## 7️⃣ Create Lambda Function (Auto Invalidation)
- Region: ap-south-1
- Runtime: Python 3.12

Lambda Code
```code
import boto3
import urllib.parse
import time

cloudfront = boto3.client('cloudfront')

DISTRIBUTION_ID = "YOUR_DISTRIBUTION_ID"

def lambda_handler(event, context):

    key = urllib.parse.unquote_plus(
        event['Records'][0]['s3']['object']['key']
    )

    print("Uploaded:", key)

    response = cloudfront.create_invalidation(
        DistributionId=DISTRIBUTION_ID,
        InvalidationBatch={
            'Paths': {
                'Quantity': 1,
                'Items': ["/" + key]
            },
            'CallerReference': str(time.time())
        }
    )

    print("Invalidation Created")

    return {
        'statusCode': 200
    }
```
Replace: YOUR_DISTRIBUTION_ID


## 8️⃣ IAM Policy for Lambda

-- Attach to Lambda execution role:
```code
{
  "Effect": "Allow",
  "Action": "cloudfront:CreateInvalidation",
  "Resource": "*"
}
```

## 9️⃣ Configure S3 Event Notification

Go to:
- S3 → Properties → Event Notifications
- Create new notification:
- Event type: All object create events
- Destination: Lambda function
- Save

## 🧪 Testing

HTTPS lock icon
Site loads via CloudFront

## Test Invalidation
- Modify index.html
- Upload new version to S3
- Go to CloudFront → Invalidations
- Confirm new Invalidation ID appears
- Refresh site
- Verify updated content loads immediately
