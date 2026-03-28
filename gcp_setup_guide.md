# Step-by-Step Guide: GCP Project Setup for Stock Management Microservices

This guide walks you through the process of setting up a Google Cloud Platform (GCP) project, enabling required APIs, and configuring a secure networking environment for your Stock Management microservices.

## Prerequisites
- You must have the [Google Cloud CLI (`gcloud`)](https://cloud.google.com/sdk/docs/install) installed and configured on your local machine.
- You must be authenticated to GCP. Run `gcloud auth login` if you haven't already.

---

## Step 1: Create the GCP Project & Set Billing

1. **Create the Project:**
   Replace `<STOCK_PROJECT_ID>` with a globally unique ID (e.g., `stock-project-12345`).
   ```bash
   gcloud projects create <STOCK_PROJECT_ID> --name="Stock Management Project"
   ```

2. **Set the Project as Default:**
   ```bash
   gcloud config set project <STOCK_PROJECT_ID>
   ```

3. **Link a Billing Account:**
   *To find your Billing Account ID, run: `gcloud alpha billing accounts list`*
   ```bash
   gcloud alpha billing projects link <STOCK_PROJECT_ID> \
       --billing-account=<YOUR_BILLING_ACCOUNT_ID>
   ```

---

## Step 2: Enable Required APIs

Enable the necessary GCP services for your microservices infrastructure:

```bash
gcloud services enable compute.googleapis.com \
    sqladmin.googleapis.com \
    dns.googleapis.com \
    storage.googleapis.com \
    firestore.googleapis.com
```

---

## Step 3: Set up Custom VPC Network

Create a secure, custom Virtual Private Cloud (VPC) for your instances.

1. **Create the VPC Network:**
   Create a custom network (this means it doesn't automatically create subnets in every region).
   ```bash
   gcloud compute networks create stock-vpc \
       --subnet-mode=custom \
       --bgp-routing-mode=regional
   ```

2. **Create the Subnet:**
   Create a subnet specifically in the `us-central1` region with the `10.0.0.0/24` CIDR block.
   ```bash
   gcloud compute networks subnets create stock-subnet \
       --network=stock-vpc \
       --range=10.0.0.0/24 \
       --region=us-central1
   ```

---

## Step 4: Configure Firewall Rules

Set up the necessary firewall rules to allow traffic based on your requirements.

1. **Allow SSH Access (Port 22):**
   *(Note: For better security in production, restrict `--source-ranges` to IAP proxies or your specific IP).*
   ```bash
   gcloud compute firewall-rules create allow-ssh \
       --network=stock-vpc \
       --allow=tcp:22 \
       --source-ranges=0.0.0.0/0
   ```

2. **Allow HTTP & Microservice Ports (80, 8080, 8761):**
   This allows external traffic to reach your Gateway, Eureka, and standard HTTP.
   ```bash
   gcloud compute firewall-rules create allow-http \
       --network=stock-vpc \
       --allow=tcp:80,tcp:8080,tcp:8761 \
       --source-ranges=0.0.0.0/0
   ```

3. **Allow Internal Communication:**
   Allow all instances within the `10.0.0.0/24` subnet to communicate freely with each other on all TCP ports.
   ```bash
   gcloud compute firewall-rules create allow-internal \
       --network=stock-vpc \
       --allow=tcp:0-65535 \
       --source-ranges=10.0.0.0/24
   ```

4. **Allow GCP Health Checks:**
   GCP Load Balancers and managed instance groups use specific IP ranges (`130.211.0.0/22` and `35.191.0.0/16`) to probe your instances' health on your application ports.
   ```bash
   gcloud compute firewall-rules create allow-health-check \
       --network=stock-vpc \
       --allow=tcp:8080-8083 \
       --source-ranges=130.211.0.0/22,35.191.0.0/16
   ```

---

## Step 5: Configure Cloud Router & Cloud NAT

Because your backend microservices (Stock, Supplier, File) should run on VMs *without* external public IPs for security, they need Cloud NAT to fetch updates, download packages, or connect to external managed APIs (like GCP Storage/Firestore).

1. **Create the Cloud Router:**
   ```bash
   gcloud compute routers create stock-router \
       --network=stock-vpc \
       --region=us-central1
   ```

2. **Create the Cloud NAT Configuration:**
   Attach a NAT translation to the router for the `stock-subnet`.
   ```bash
   gcloud compute routers nats create stock-nat \
       --router=stock-router \
       --region=us-central1 \
       --auto-allocate-nat-external-ips \
       --nat-all-subnet-ip-ranges
   ```

---

## Step 6: Create Cloud SQL Database for Stock Service

1. **Create the MySQL Instance:**
   ```bash
   gcloud sql instances create stock-mysql \
       --database-version=MYSQL_8_0 \
       --tier=db-f1-micro \
       --region=us-central1 \
       --root-password=<YOUR_ROOT_PASSWORD> \
       --authorized-networks=0.0.0.0/0
   ```
   *(Note: `0.0.0.0/0` allows access from anywhere, which is useful for testing but should be restricted in a real production environment.)*

2. **Create the Database:**
   ```bash
   gcloud sql databases create stockdb \
       --instance=stock-mysql
   ```

3. **Get the Public IP Address:**
   Run the following command and note the IP address of your instance:
   ```bash
   gcloud sql instances describe stock-mysql --format="value(ipAddresses[0].ipAddress)"
   ```

### Updating the Stock Service for Production

To update the `stock-service` to use this Cloud SQL instance instead of `localhost`, update your Config Server configuration repo. Open [stock-config-repo/stock-service/application.yml](file:///Users/supunmadhuranga/Desktop/CL/stock-main/stock-config-repo/stock-service/application.yml) and add/update the `spring.datasource` properties:

```yaml
spring:
  application:
    name: stock-service
  datasource:
    url: jdbc:mysql://<CLOUD_SQL_PUBLIC_IP>:3306/stockdb?useSSL=false&serverTimezone=UTC&allowPublicKeyRetrieval=true
    username: root
    password: <YOUR_ROOT_PASSWORD>
```
When you deploy your `stock-service` to a GCP VM, it will securely fetch this configuration and connect directly to your newly created Cloud SQL instance!

---

## Summary

Your GCP environment is now prepped! You have:
- A dedicated project with billing activated and required APIs enabled.
- A custom VPC (`stock-vpc`) and subnet (`stock-subnet`) for network isolation.
- Firewall rules configured for SSH, public gateway access, internal microservice chatter, and health checks.
- Cloud NAT enabled so your secure internal VMs can still access the internet for updates and GCP APIs.
- A managed Cloud SQL MySQL 8.0 instance ready for the Stock Service.
