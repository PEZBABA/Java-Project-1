Mercury: Laundry & Dry-Cleaning Management System on AWS

Mercury (mercury.war) is a Java web application (JSP + Servlets) for running a laundry and dry-cleaning business: customer records and CRM, invoicing and pick-up, payments, expenses, payroll, accounting reports and branch management. It is deployed here on a highly available AWS architecture using CentOS servers, an Auto Scaling Group, an Application Load Balancer and CloudFront with a custom HTTPS domain.

Architecture
                        Users
                          │
                  ┌───────▼────────┐
                  │  Route 53 DNS  │   peterbaba.name.ng (registered at Go54)
                  └───────┬────────┘
                          │  Alias record
                  ┌───────▼────────┐
                  │   CloudFront   │   HTTPS (ACM cert, us-east-1)
                  └───────┬────────┘
                          │  Origin
                  ┌───────▼────────┐
                  │      ALB       │   Sticky sessions enabled
                  └───────┬────────┘
                          │
                  ┌───────▼────────┐
                  │  Target Group  │   Health checks
                  └───┬────────┬───┘
                      │        │
   ┌──────────────────▼─┐   ┌──▼──────────────────┐
   │ AZ-1               │   │ AZ-2                │
   │ Public Subnet A    │   │ Public Subnet B     │
   │ EC2 #1 (CentOS)    │   │ EC2 #2 (CentOS)     │
   │ Tomcat + mercury   │   │ Tomcat + mercury    │
   └─────────┬──────────┘   └──────────┬──────────┘
             └────────────┬────────────┘
                   MySQL database
        Managed by an Auto Scaling Group
        inside a custom VPC (4 subnets, 2 AZs)
Tech Stack
Layer	Technology
Application	Java, JSP, Servlets (packaged as mercury.war)
App server	Apache Tomcat on CentOS
Database	MySQL (JDBC driver mysql-connector-java-5.1.6 bundled in the WAR)
Compute	EC2 (CentOS) in an Auto Scaling Group
Load balancing	Application Load Balancer + Target Group
Network	Custom VPC, 4 subnets, 2 Availability Zones
DNS / Domain	Go54 (registrar) + Route 53 (DNS hosting)
TLS / CDN	ACM certificate + CloudFront
Application Overview

Based on the contents of the WAR:

Customer & CRM: customer registration, preferences, loyalty/discounts, customer care, customer login portal
Operations: order intake, invoices, pick-up and delivery status, tag and barcode printing, overdue tracking
Finance & accounting: payments, deposits, expenses, loans, fixed assets, chart of accounts, journal, income statements, cash flow
Staff & payroll: staff records, payroll, pension, pay records
Procurement: purchases, vendors, stores, consumables
Admin: multi-branch setup, backups, email/SMS notifications, user manual (easymanual.pdf)
mercury.war
├── *.jsp                 # ~500 pages (UI + report pages)
├── images/, *.js, *.css  # static assets
├── easymanual.pdf        # user manual
├── META-INF/
└── WEB-INF/
    ├── classes/          # compiled servlets and helpers (Classespack, com.vasworks, newpackage)
    ├── lib/              # MySQL JDBC driver
    └── glassfish-web.xml # original context root: /EasyWashCS
Network Layout
VPC: custom CIDR (e.g. 10.0.0.0/16) with an Internet Gateway
Subnets: 4 across 2 AZs
Public Subnet A (AZ-1): EC2 instance 1
Public Subnet B (AZ-2): EC2 instance 2
Two additional subnets (one per AZ), reserved for private resources such as the database
Security groups:
ALB SG: 80/443 from the internet
EC2 SG: 8080 only from the ALB SG; SSH (22) only from your IP
Database SG: 3306 only from the EC2 SG
Deployment Steps
1. Network
Create the VPC and attach an Internet Gateway.
Create 4 subnets across 2 AZs (2 public, 2 private).
Add a public route table (0.0.0.0/0 to IGW) and associate it with the public subnets.
2. Database

Create a MySQL database (RDS in the private subnets, or MySQL on a server) and import the application schema. Update the database host, name and credentials used by the application's connector classes (newpackage/SQLconnector*), then rebuild or repackage the WAR if they are compiled in. Never commit credentials to GitHub.

3. App server (CentOS)

The WAR was built with Java 7 and old libraries, so use Tomcat 9 (not 10, which uses the jakarta namespace and will break this app) with Java 8 or later, and test thoroughly.

bash
sudo yum update -y
sudo yum install -y java-1.8.0-openjdk tomcat   # or install Tomcat 9 manually from tomcat.apache.org
sudo systemctl enable --now tomcat

# Deploy as the root application so the site opens at /
sudo systemctl stop tomcat
sudo rm -rf /var/lib/tomcat/webapps/ROOT
sudo cp mercury.war /var/lib/tomcat/webapps/ROOT.war
sudo systemctl start tomcat

Check locally with curl -I http://localhost:8080/. Paths vary by install method, so adjust /var/lib/tomcat if you installed Tomcat elsewhere (for example /opt/tomcat). Once it works, create an AMI (or use Launch Template user data) so new instances come up ready.

4. Load balancing and auto scaling
Create a Target Group (HTTP :8080, health check path /).
Enable stickiness on the Target Group (load-balancer cookie). The app uses login sessions that are stored per server, so without stickiness users get logged out when the ALB switches instances.
Create an internet-facing ALB across Public Subnet A and B, forwarding to the Target Group.
Create a Launch Template from the AMI.
Create an Auto Scaling Group across both public subnets, attached to the Target Group (desired 2, min 2, max as needed, CPU-based scaling policy).
5. Domain, DNS and HTTPS
Create a Route 53 hosted zone for the domain.
At Go54, replace the nameservers with the 4 NS records from the hosted zone.
Request a public certificate in ACM in us-east-1 (required for CloudFront) and validate via DNS.
Create a CloudFront distribution:
Origin: the ALB DNS name
Alternate domain names: peterbaba.name.ng, www.peterbaba.name.ng
Custom SSL certificate: the ACM cert
Viewer protocol policy: Redirect HTTP to HTTPS
Cache policy: CachingDisabled, origin request policy that forwards all cookies, headers and query strings, and allowed methods GET, HEAD, OPTIONS, PUT, POST, PATCH, DELETE (this is a dynamic, login-based app)
In Route 53, create A (Alias) records for the root domain and www pointing to the distribution.
6. Test
Open https://peterbaba.name.ng, confirm the padlock and the login page.
Log in, move between pages and confirm you stay logged in (stickiness working).
Stop one instance and confirm the site stays up while the ASG replaces it.
Confirm both targets are healthy in the Target Group.
Configuration Notes
Context path: the original descriptor sets /EasyWashCS. Deploying as ROOT.war serves the app at /. If you deploy as mercury.war it will be at /mercury/, and the ALB health check path and CloudFront origin path must match.
Email/SMS: the app includes email and SMS notification classes. Configure SMTP/SMS gateway settings for your environment.
Time zone: set the servers to Africa/Lagos for correct invoice and report dates.
Security Practices
EC2 only accepts app traffic from the ALB; database only from the app servers
HTTPS enforced at CloudFront with an ACM certificate
SSH restricted to a trusted IP
Database credentials kept out of source control (use environment config or AWS Secrets Manager)
Known Limitations and Improvements
Old dependencies (Java 7 build, MySQL driver 5.1.6): upgrade the driver and runtime when possible
Move EC2 instances to private subnets behind a NAT Gateway
Add AWS WAF on CloudFront
Share sessions across servers (Tomcat session replication or ElastiCache) instead of relying on stickiness
Infrastructure as Code with Terraform, and CI/CD with GitHub Actions
CloudWatch alarms for ALB, ASG and database metrics
Credits
Application: Mercury / EasyWash laundry management system
AWS infrastructure & deployment: [Your name]
License

All rights reserved. Update this section to match the application owner's terms.
