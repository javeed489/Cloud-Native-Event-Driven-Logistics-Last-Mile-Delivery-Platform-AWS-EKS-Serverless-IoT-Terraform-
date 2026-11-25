# Cloud-Native-Event-Driven-Logistics-Last-Mile-Delivery-Platform-AWS-EKS-Serverless-IoT-Terraform-
flowchart TB
    subgraph Clients["Clients & Edge Devices"]
        Driver["Driver Handheld / Mobile App"]
        Web["Dispatcher Portal"]
        Van["GPS IoT Trackers (MQTT)"]
        PG["Payment Gateway (Moyasar / Tap / Stripe)"]
    end

    subgraph Edge["AWS Edge & Public Ingress"]
        WAF["AWS WAF"]
        APIGW["Amazon API Gateway"]
        IoT["AWS IoT Core (X.509 + Basic Ingest)"]
    end

    subgraph VPC["VPC (Multi-AZ: AZ-a & AZ-b)"]
        VPCLink["VPC Link v2"]
        
        subgraph PrivateApp["Private App Subnets (AZ-a / AZ-b)"]
            ALB["Internal ALB (AWS Load Balancer Controller)"]
            subgraph EKS["Amazon EKS Cluster (EC2 Nodes + Karpenter)"]
                Pod1["Orders & Dispatch Pods (HPA + PDB)"]
                Pod2["Payment & COD Pods (EKS Pod Identity)"]
            end
        end

        subgraph PrivateData["Private Data Subnets (No Internet)"]
            RDS[("Amazon RDS PostgreSQL (Multi-AZ)")]
            VPCE["VPC Endpoints (S3, ECR, Secrets Manager, EKS Auth)"]
        end
    end

    subgraph Serverless["Serverless, Storage & Analytics"]
        LambdaPoD["Lambda: PoD Pre-Signed URL & Webhooks"]
        SQS["SQS DLQ"]
        Loc["Amazon Location Service (Geofences & Routes)"]
        EB["Amazon EventBridge"]
        SNS["Amazon SNS (SMS / Email Alerts)"]
        DDB[("Amazon DynamoDB (Live GPS State)")]
        S3PoD[("Amazon S3 (pod/ 60-Day Lifecycle)")]
        Firehose["Data Firehose -> S3 Parquet -> Glue -> Athena -> QuickSight"]
        ECR["Amazon ECR (Docker Images)"]
        SM["AWS Secrets Manager"]
    end

    Driver & Web & PG --> WAF --> APIGW
    APIGW --> VPCLink --> ALB
    ALB -->|"Direct Pod IP Routing"| Pod1 & Pod2
    APIGW -->|"/pod/upload-url & /webhooks"| LambdaPoD
    LambdaPoD --> S3PoD & SQS
    Pod1 & Pod2 --> RDS
    Pod1 & Pod2 -.-> VPCE -.-> SM & ECR & S3PoD

    Van --> IoT --> Loc --> EB --> SNS
    IoT --> DDB
    IoT --> Firehose
