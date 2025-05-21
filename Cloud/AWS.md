
## AWS Global Architecture

### 1. **Regions**

- A **Region** is a physical location in the world where AWS has **data centers**.
    
- Example: `us-east-1` (N. Virginia), `ap-south-1` (Mumbai).
    

### 2. **Availability Zones (AZs)**

- Each Region has **2+ isolated Availability Zones**, which are **physically separate data centers**.
    
- Ensures **high availability & fault tolerance**.
    

### 3. **Edge Locations**

- Used for **content delivery via CloudFront**.
    
- Caches data closer to users, reducing latency.


## Major Categories of AWS Services (With Examples)

| Category                       | Services                                        | Purpose                               |
| ------------------------------ | ----------------------------------------------- | ------------------------------------- |
| 🧠 **Compute**                 | EC2, Lambda, ECS, EKS, Elastic Beanstalk        | Run applications and workloads        |
| 🗄️ **Storage**                | S3, EBS, EFS, Glacier                           | Store and retrieve data               |
| 🛢️ **Databases**              | RDS, DynamoDB, Aurora, Redshift                 | Manage relational and NoSQL databases |
| 🌐 **Networking & CDN**        | VPC, Route 53, API Gateway, CloudFront          | Connect and deliver content globally  |
| 🔐 **Security & Identity**     | IAM, KMS, Cognito, Shield, WAF                  | Control access and protect data       |
| 🛠️ **Developer Tools**        | CodeCommit, CodeBuild, CodeDeploy, CodePipeline | CI/CD & DevOps                        |
| 📊 **Monitoring & Management** | CloudWatch, CloudTrail, Trusted Advisor, Config | Monitor resources and compliance      |
| 🤖 **AI/ML**                   | SageMaker, Rekognition, Comprehend, Lex         | Build intelligent applications        |
| 💬 **Messaging**               | SQS, SNS, EventBridge, MQ                       | Decouple and connect applications     |
| 🧭 **Migration & Transfer**    | DMS, Snowball, Migration Hub                    | Move data/applications to AWS         |

## Cheat Sheet Summary

| Layer          | Services                          |
| -------------- | --------------------------------- |
| **Frontend**   | Route 53, CloudFront, API Gateway |
| **Compute**    | EC2, Lambda, Elastic Beanstalk    |
| **Data**       | RDS, DynamoDB, Redshift           |
| **Storage**    | S3, EBS, Glacier                  |
| **Networking** | VPC, ELB, NAT Gateway             |
| **Security**   | IAM, Shield, WAF                  |
| **Monitoring** | CloudWatch, CloudTrail            |
|                |                                   |

AWS Lambda --> Is server-less compute service that lets you run code without provisioning or managing servers.
				Just upload your code and AWS lambda takes care of every thing else. like scaling, fault tolerance and 
				availability.



## Key Features of AWS Lambda

|Feature|Description|
|---|---|
|**Event-driven**|Lambda runs code in response to events (e.g., S3 upload, API Gateway call, DynamoDB stream).|
|**Serverless**|No need to manage servers — AWS handles scaling and infrastructure.|
|**Auto-scaling**|Automatically scales out to handle the load, and scales in when idle.|
|**Pay-as-you-go**|Charged based on number of requests and execution time (in milliseconds).|
|**Stateless**|Each execution is independent; use external services (like S3, RDS) for state/data.|
```
User → API Gateway → Lambda → DynamoDB/S3
                 ↑
             Event Trigger
```


_"How does AWS ensure scalability and fault tolerance?"_, you can answer:

> "AWS uses multiple Availability Zones per region, load balancers, auto-scaling groups, and global edge locations via CloudFront to distribute traffic and ensure fault tolerance and low latency."


#### **Classes of S3 bucket**

- S3 Standard
    
- S3 Intelligent-Tiering
    
- S3 Standard-IA (Infrequent Access)
    
- S3 One Zone-IA
    
- S3 Glacier
    
- S3 Glacier Deep Archive
