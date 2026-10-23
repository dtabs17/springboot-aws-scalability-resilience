# Spring Boot AWS Scalability and Resilience

An investigation into how horizontal scaling affects the performance and resilience of a Spring Boot Product Service deployed on AWS.

The system has two REST services: **Product Service** and **Coupon Service**. The focus of the project was deploying and scaling Product Service, measuring its behaviour under load, and observing what happens when an EC2 instance fails.

## Architecture

- **Product Service:** Spring Boot application on EC2, behind an Application Load Balancer (ALB) and managed by an Auto Scaling Group (ASG) across two Availability Zones.
- **Coupon Service:** A separate Spring Boot application on a single EC2 instance, called by Product Service when creating a product with a coupon.
- **Database:** Amazon RDS MySQL, with separate logical `productdb` and `coupondb` databases on one Single-AZ DB instance.
- **Deployment:** Executable Java 17 JARs managed by `systemd`; a custom AMI and launch template provisioned Product instances.
- **Configuration:** AWS Systems Manager Parameter Store and instance IAM roles for Product runtime configuration.
- **Monitoring and testing:** Amazon CloudWatch and Grafana k6.

The AWS environment was built for the investigation and is not maintained as a public live service.

## Experiments and results

### 1. One versus two Product instances

Grafana k6 sent GET requests to `/productapi/products` through the ALB at five fixed arrival rates. Each run lasted two minutes, with ASG capacity fixed at one instance and then two instances.

| Target RPS | Achieved RPS (1 / 2 instances) | p95 latency, ms (1 / 2) | Dropped iterations (1 / 2) |
| ---: | ---: | ---: | ---: |
| 30 | 30.00 / 29.97 | 44.47 / 49.65 | 0 / 0 |
| 100 | 99.94 / 99.98 | 42.54 / 50.09 | 0 / 0 |
| 200 | 199.96 / 199.96 | 45.20 / 62.88 | 0 / 0 |
| 400 | 399.43 / 399.60 | 55.33 / 34.66 | 57 / 38 |
| 600 | 598.06 / 580.68 | 28.23 / 100.20 | 199 / 1,220 |

At **400 RPS**, the two-instance configuration recorded about **37% lower p95 latency** in the measured runs. At lighter loads, a second instance did not demonstrate a clear benefit. The initial 600 RPS comparison was affected by k6 virtual-user limits and changes to the client network during follow-up tests, so it does not support a firm conclusion about AWS-side performance.

A *dropped iteration* is a scheduled k6 iteration that did not start; it is not a request that received an HTTP error. All HTTP requests completed during these fixed-capacity runs returned HTTP 200.

### 2. Automatic scaling

A target-tracking policy used **3,000 ALB requests per target per minute**, with an ASG capacity range of **1 to 3 instances**. Under a 150 RPS workload lasting 12 minutes, the ASG scaled **1 to 3 instances**. After the test, it scaled back to **1**. The policy threshold was selected to make elasticity observable, not as a production-validated capacity limit.

### 3. Controlled failure and recovery

With two healthy Product instances, k6 sent 100 RPS for 15 minutes while one EC2 instance was deliberately terminated. AWS launched a replacement and healthy capacity returned to two instances in approximately **1 to 2 minutes**.

However, **1,026 of 89,181 completed requests (1.15%) failed**. CloudWatch also recorded 1,026 ALB-generated 5XX responses around the termination. Automatic replacement worked, but the client experience was not uninterrupted.

## Engineering decisions and limitations

- **JAR deployment instead of Docker:** Kept the EC2-focused experiment simpler, at the expense of container-based runtime consistency and portability.
- **Custom AMI:** Improved repeatability of instance launches, but application and Java updates require a new image.
- **Request-based Auto Scaling:** Better suited to demonstrating elasticity in these tests than CPU-based scaling, since observed average CPU was relatively low.
- **Shared Single-AZ RDS and one Coupon EC2:** Reduced costs and kept the investigation focused on Product Service, but left important shared failure risks.
- **Public application networking and HTTP:** Appropriate only for this controlled investigation; a production deployment would need stronger network isolation and HTTPS.

The load tests were short, most conditions were tested once, and the client connection was not consistent across all runs. The GET workload also did not exercise coupon-backed product creation under load. The results should therefore be treated as an investigation, not a production capacity benchmark.

**Next experiment:** Compare planned instance removal with an abrupt failure, reviewing ALB health checks, traffic routing and connection draining. Repeat the same failure workload and compare 5XX errors and recovery behaviour.

## Repository structure

```text
productservice/    Spring Boot Product API
couponservice/     Spring Boot Coupon API
```

The AWS infrastructure was configured manually for the investigation. This repository documents the architecture and measured behaviour rather than providing infrastructure-as-code for automatic redeployment.

## Running locally

Prerequisites: **Java 17**, **Maven**, and **MySQL**. The Maven projects also use a Java 17 toolchain configuration.

1. Create two MySQL databases named `productdb` and `coupondb`.
2. In each service folder, create a local `.env` file next to `pom.xml`. These files are intentionally excluded from Git.
3. Supply the following values for Coupon Service:

   ```properties
   DB_URL=jdbc:mysql://localhost:3306/coupondb
   DB_USERNAME=your_local_mysql_user
   DB_PASSWORD=your_local_mysql_password
   ```

4. Supply the following values for Product Service:

   ```properties
   DB_URL=jdbc:mysql://localhost:3306/productdb
   DB_USERNAME=your_local_mysql_user
   DB_PASSWORD=your_local_mysql_password
   COUPON_SERVICE_URL=http://localhost:9091/couponapi/coupons/
   ```

5. Start Coupon Service, then Product Service, from their respective project folders:

   ```bash
   mvn spring-boot:run
   ```

Coupon Service listens on port **9091** and Product Service on **9090**. Example endpoints:

- `GET http://localhost:9091/couponapi/coupons`
- `GET http://localhost:9091/couponapi/coupons/{code}`
- `GET http://localhost:9090/productapi/products`

Product creation uses `POST /productapi/products` and requires an existing coupon code in the current implementation.

## Project context

Developed for a university cloud scalability and resilience investigation, using Spring Boot starter services adapted for the AWS deployment and experiments. The temporary AWS resources will be decommissioned after evidence collection to avoid ongoing charges.
