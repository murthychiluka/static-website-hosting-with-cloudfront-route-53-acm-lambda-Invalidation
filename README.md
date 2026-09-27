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
*************************************************************************************************************

```text
what happens internally when you hit a cloudfront url?

When you hit a CloudFront URL, for example:

https://d123abc.cloudfront.net/index.html

a lot happens behind the scenes. The easiest way to understand it is as a request flowing from the user → DNS → CloudFront edge → cache → origin → back to the user.

Overall flow
User / Browser
      │
      │ HTTPS request
      ▼
     DNS
      │
      │ CloudFront IP
      ▼
CloudFront Edge Location
      │
      │
      ├── Cache HIT ───────────────► Return response
      │
      │
      └── Cache MISS
              │
              ▼
           Origin
      (S3 / ALB / EC2 / API)
              │
              ▼
        CloudFront Edge
              │
              ▼
          User/Browser

```
```text
Let's go step by step.

1. You enter the CloudFront URL

For example:

https://d123abc.cloudfront.net/images/logo.png

The browser first needs to find where:

d123abc.cloudfront.net

is located.

```
```text

2. DNS resolution happens

The browser asks DNS:

What IP address should I use for
d123abc.cloudfront.net?

CloudFront uses DNS to direct the client toward an appropriate CloudFront edge location.

Conceptually:

Browser
   │
   ▼
DNS Resolver
   │
   ▼
CloudFront DNS
   │
   ▼
Appropriate CloudFront edge

The actual edge selection considers things such as network proximity and CloudFront's routing infrastructure.

3. TCP connection is established

For HTTPS, the client establishes a connection to the CloudFront edge.

Conceptually:

Client
  │
  │ TCP connection
  ▼
CloudFront Edge

With HTTP/2 or HTTP/3, the exact transport can differ; HTTP/3 uses QUIC rather than traditional TCP.
```
```text

4. TLS handshake happens

Because you're accessing:

https://...

TLS negotiation occurs.

The browser validates the CloudFront certificate.

For example:

Browser
   │
   │ TLS handshake
   ▼
CloudFront
   │
   ▼
Certificate validation

CloudFront can use an ACM certificate associated with the distribution for your custom domain.

For example:

https://www.example.com

could point to:

CloudFront Distribution
        │
        └── ACM certificate
5. CloudFront receives the HTTP request

Now CloudFront receives something like:

GET /images/logo.png HTTP/2
Host: d123abc.cloudfront.net

CloudFront determines which distribution, behavior, cache policy, origin request policy, and other configuration apply to the request.

For example, you might have:

/images/*       → S3
/api/*          → ALB
/static/*       → S3
6. CloudFront checks its cache

This is one of the most important steps.

CloudFront asks:

"Do I already have this object cached at this edge location?"

For example:

GET /images/logo.png
Cache HIT

If the object exists and is still valid:

Browser
   │
   ▼
CloudFront Edge
   │
   │ Cache HIT
   ▼
Cached object
   │
   ▼
Browser

The origin isn't contacted.

This is why CloudFront can significantly reduce latency and origin load.
```

```text

7. What determines whether it is a cache hit?

It's not simply the URL alone.

CloudFront's caching behavior depends on the distribution configuration, including things such as:

URL/path
HTTP method
cache policy
selected headers
cookies
query strings
TTL
cache-control headers

For example, depending on your cache policy:

/products?id=10

and

/products?id=20

could be treated as different cache keys.

8. Cache MISS
```

```text

Suppose CloudFront doesn't have the requested object:

Cache MISS

Then CloudFront needs to retrieve it from the configured origin.

The origin could be:

CloudFront
   │
   ├── S3
   ├── ALB
   ├── EC2
   ├── API Gateway
   └── Custom HTTP origin

For example:

CloudFront
     │
     ▼
Application Load Balancer
     │
     ▼
EKS Service
     │
     ▼
Pod
```

```text
9. CloudFront connects to the origin

Let's say your architecture is:

User
  │
  ▼
CloudFront
  │
  ▼
ALB
  │
  ▼
EKS
  │
  ▼
Pod

CloudFront sends the request to the origin.

The ALB receives:

GET /api/products

and routes it to an appropriate target.
```
```text
10. Application processes the request

Your application might then:

Pod
 │
 ├── Validate request
 ├── Query database
 ├── Generate response
 └── Return JSON

For example:

{
  "products": [
    {
      "id": 101,
      "name": "Laptop"
    }
  ]
}

The response travels back:

Pod
 │
 ▼
ALB
 │
 ▼
CloudFront
```
```text
11. CloudFront may cache the response

For cacheable content, CloudFront can store the origin response at the edge.

Origin
   │
   ▼
CloudFront Edge
   │
   ├── Store in cache
   │
   ▼
User

Then the next request may be:

User
  │
  ▼
CloudFront
  │
  └── CACHE HIT
          │
          ▼
        User

The origin doesn't need to process that request again.
```

```text

12. Response reaches your browser

Finally:

CloudFront
    │
    ▼
Browser

The browser receives:

HTTP/2 200 OK
Content-Type: image/png
...

and renders the content.

Complete flow

Put everything together:

                1. Request
Browser ───────────────────────────►
                                     
                2. DNS
Browser ───────► DNS ────────────────►
                                     
                3. CloudFront Edge
Browser ───────────────────────────► CloudFront
                                      │
                                      │ 4. Cache lookup
                                      │
                            ┌─────────┴─────────┐
                            │                   │
                         HIT                   MISS
                            │                   │
                            ▼                   ▼
                       Cached data          Origin
                            │                   │
                            │              ALB / S3
                            │                   │
                            │                   ▼
                            │              Application
                            │                   │
                            │                   ▼
                            │              Response
                            │                   │
                            │◄──────────────────┘
                            │
                            ▼
                         Browser
Where does Route 53 fit?

If you're using a custom domain:

https://www.example.com

you might have:

Route 53
    │
    ▼
CloudFront

For example:

www.example.com
       │
       ▼
Route 53
       │
       ▼
CloudFront Distribution
       │
       ▼
ALB
       │
       ▼
EKS

So Route 53 doesn't serve the application. It helps resolve the domain to CloudFront.

Where does WAF fit?

If AWS WAF is associated with CloudFront:

User
  │
  ▼
CloudFront
  │
  ▼
AWS WAF
  │
  ├── BLOCK → Request rejected
  │
  └── ALLOW
        │
        ▼
     Cache
        │
        ▼
      Origin

So WAF can stop malicious requests before they reach your origin.

Important interview point: CloudFront doesn't always contact the origin

This is a very common interview question.

Cache HIT
User
 ↓
CloudFront
 ↓
Cache
 ↓
User

Origin is not contacted.

Cache MISS
User
 ↓
CloudFront
 ↓
Origin
 ↓
CloudFront
 ↓
User

This distinction is fundamental.

Example with your EKS architecture

If you have:

Internet
    │
    ▼
CloudFront
    │
    ▼
ALB
    │
    ▼
EKS
    │
    ▼
Ingress / Service
    │
    ▼
Pods
```
```text
a request might look like:

https://www.example.com/api/books

Flow:

1. Browser
      ↓
2. DNS / Route 53
      ↓
3. CloudFront Edge
      ↓
4. WAF inspection
      ↓
5. Cache lookup
      ↓
6. Cache MISS
      ↓
7. ALB
      ↓
8. EKS Service / Ingress
      ↓
9. Backend Pod
      ↓
10. Database
      ↓
11. Backend Pod
      ↓
12. ALB
      ↓
13. CloudFront
      ↓
14. Browser

For a subsequent cacheable request:

Browser
   ↓
CloudFront
   ↓
CACHE HIT
   ↓
Browser

The request doesn't need to travel all the way to EKS.

Interview one-liner

"When a user accesses a CloudFront URL, DNS resolves the CloudFront distribution, the client establishes an HTTPS connection with an edge location, and CloudFront evaluates the request against its distribution and cache configuration. If the object is cached, CloudFront returns it directly. On a cache miss, CloudFront forwards the request to the configured origin such as S3, ALB, or an application, receives the response, potentially caches it according to the cache policy and TTL, and returns it to the client."
```
