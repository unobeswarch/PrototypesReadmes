# 🚀 Delivery: Prototype 4 - AWS Deployment Architecture
**Software Architecture** | Universidad Nacional de Colombia 🎓

---

## 👥 Team 1B

| **Member** | **Email** |
|------------|-----------|
| 🔹 Edinson Sanchez Fuentes | edsanchezf@unal.edu.co |
| 🔹 Adrian Ramirez Gonzalez | adramirez@unal.edu.co |
| 🔹 Sergio Nicolas Siabatto Cleves | ssiabatto@unal.edu.co |
| 🔹 Martin Polanco Barrero | mpolancob@unal.edu.co |
| 🔹 David Fernando Adames Rondon | dadames@unal.edu.co |
| 🔹 Julian Esteban Mendoza Wilches | jmendozaw@unal.edu.co |

## NeumoDiagnostics
<div align="center">

![Team Logo](./images/logo.PNG)

</div>

---

## 🩺 Software System: **NeumoDiagnostics**

### 📋 Overview
**NeumoDiagnostics** is an AI-powered support platform designed to assist doctors in reviewing patient radiographs for pneumonia detection. Our system integrates advanced machine learning with comprehensive patient management features.

> ⚠️ **Important Note**: This model is designed to support, not replace, medical judgment. The final diagnosis always remains with the healthcare professional.

---

## 🏗️ **Architectural Structures**

Our NeumoDiagnostics system employs multiple architectural views to ensure comprehensive documentation and understanding of the system's design. Each view provides unique insights into different aspects of the architecture.

---

### 🔗 **Component and Connector (C&C) Structure**

#### 📊 **C&C View**
*Visual representation of system components and their interconnections in AWS cloud environment*

<div align="center">

![C&C View](./images/cycview.png)

</div>

#### **🎯 Description of Architectural Elements and Relations:**

This view describes runtime components deployed on AWS, the interfaces they provide/require, and the connectors between them. It focuses on communication paths, protocols, and AWS-managed services.

**User Interfaces:**
- `web browser`: Web-based user interface accessing the system through HTTPS
  - Connectors: HTTPS to `Application Load Balancer`
- `mobile app` (future): Mobile application for healthcare professionals
  - Connectors: HTTPS to `Application Load Balancer`
- `external systems`: Third-party healthcare systems integrating via API
  - Connectors: HTTPS-REST/GraphQL to `Application Load Balancer`

**Edge Layer:**
- `Route 53`: DNS service for domain name resolution
  - Connectors: DNS queries from clients
- `CloudFront` (optional): CDN for static content delivery
  - Connectors: HTTPS to clients, origin requests to ALB
- `AWS WAF`: Web application firewall for security
  - Connectors: Inspects traffic to/from ALB

**Load Balancing and Routing:**
- `Application Load Balancer`: Single entry point for all HTTP/HTTPS traffic
  - Provided interfaces: HTTPS:443 (terminates SSL/TLS), HTTP:80 (redirects to HTTPS)
  - Required connectors: HTTP to ECS services (web-frontend, api-gateway)
  - Target Groups: `web-frontend-tg`, `api-gateway-tg`
  - Functions: SSL/TLS termination, path-based routing, health checks, Multi-AZ distribution

**Frontend Services (ECS Fargate):**
- `web-frontend` [2-6 tasks]: Next.js application for user interface
  - Provided interfaces: HTTP:3000
  - Required connectors: HTTPS-REST and HTTP-GRAPHQL to `api-gateway`
  - Auto-scaling: Based on CPU, memory, and request count
  
**API Gateway and Orchestration (ECS Fargate):**
- `api-gateway` [3-9 tasks]: Go-based API Gateway with service discovery
  - Provided interfaces: HTTP:8080 (REST endpoints, GraphQL endpoint at `/graphql`)
  - Required connectors: 
    - HTTP-REST to `auth-be`, `prediagnostic-be`, `message-producer`
    - Service discovery queries to `AWS Cloud Map`
  - Functions: Request validation, composition, orchestration, authentication middleware
  - Auto-scaling: Weighted distribution across tasks

**Backend Services (ECS Fargate):**
- `auth-be` [2-6 tasks]: Go-based authentication and user management service
  - Provided interfaces: HTTP:8081 (login, logout, registration, profile management)
  - Required connectors: 
    - PostgreSQL driver to `RDS PostgreSQL`
    - AWS SDK to `S3` (profile images bucket)
    - AWS Secrets Manager (database credentials, JWT secrets)
  - Service Discovery: Registered in `AWS Cloud Map` as `auth-be.neumodiagnostics.local`

- `prediagnostic-be` [2-6 tasks]: Python-based diagnostic and ML inference service
  - Provided interfaces: HTTP:8000 (radiograph upload, prediction, case management, diagnosis registration)
  - Required connectors:
    - MongoDB driver to `DocumentDB`
    - AWS SDK to `S3` (radiography images bucket)
    - AWS Secrets Manager (database credentials)
  - Service Discovery: Registered in `AWS Cloud Map` as `prediagnostic-be.neumodiagnostics.local`

- `message-producer` [2-4 tasks]: Go-based message publisher
  - Provided interfaces: HTTP:8082 (notification requests from api-gateway)
  - Required connectors:
    - AMQP to `Amazon MQ`
    - AWS SDK to `S3` (temporary storage)
  - Service Discovery: Registered in `AWS Cloud Map` as `message-producer.neumodiagnostics.local`

- `notification-be` [2-4 tasks]: Go-based asynchronous notification consumer
  - Provided interfaces: Background consumer (no HTTP interface)
  - Required connectors:
    - AMQP subscription to `Amazon MQ`
    - AWS SES SDK (email sending)
    - AWS Secrets Manager (SES credentials)
  - Functions: Consumes notification messages, sends emails via SES

**Data Stores and Managed Services:**
- `RDS PostgreSQL` (Multi-AZ): Relational database for authentication and user data
  - Provided interfaces: PostgreSQL protocol on port 5432
  - Configuration: Multi-AZ deployment with automatic failover
  - Accessed exclusively by: `auth-be`
  - Replication: Synchronous replication to standby in different AZ

- `DocumentDB` (Cluster): MongoDB-compatible database for clinical documents
  - Provided interfaces: MongoDB protocol on port 27017
  - Configuration: 1 writer + 2 reader instances across 3 AZs
  - Accessed exclusively by: `prediagnostic-be`
  - Replication: Cluster with automatic failover

- `Amazon MQ` (RabbitMQ Cluster): Message broker for asynchronous communication
  - Provided interfaces: AMQP on port 5672, Management UI on port 15672
  - Configuration: Active/Standby deployment across 2 AZs
  - Accessed by: `message-producer` (publisher), `notification-be` (consumer)

- `Amazon S3` (Multiple Buckets): Object storage for images and files
  - `neumo-radiography-images`: Patient X-ray images (accessed by `prediagnostic-be`)
  - `neumo-profile-images`: User profile pictures (accessed by `auth-be`)
  - `neumo-backup-data`: Database backups
  - `neumo-alb-logs`: Load balancer access logs
  - Provided interfaces: S3 API over HTTPS
  - Features: Versioning, lifecycle policies, encryption at rest (SSE-S3/SSE-KMS)

- `Amazon SES`: Email service for notifications
  - Provided interfaces: SMTP (port 587), SES API
  - Accessed by: `notification-be`
  - Features: Bounce handling, complaint tracking

**Service Discovery and Monitoring:**
- `AWS Cloud Map`: Service registry for ECS service discovery
  - Namespace: `neumodiagnostics.local`
  - Registered services: api-gateway, auth-be, prediagnostic-be, message-producer
  - DNS-based discovery with automatic registration/deregistration

- `CloudWatch`: Monitoring and logging service
  - Metrics: ECS Container Insights, RDS Performance Insights, ALB metrics
  - Logs: Container logs, VPC Flow Logs, CloudTrail audit logs
  - Alarms: CPU, memory, error rates, unhealthy targets

- `AWS X-Ray`: Distributed tracing service
  - Traces requests across: ALB → api-gateway → backend services
  - Provides: Service map, latency analysis, error detection

**Security Services:**
- `AWS Secrets Manager`: Secure credential storage
  - Stores: Database passwords, API keys, JWT secrets, SES credentials
  - Features: Automatic rotation, encryption with KMS

- `AWS KMS`: Key management service
  - Encrypts: RDS data, DocumentDB data, S3 objects, Secrets Manager secrets

**Connector Summary and Directionality:**
- HTTPS: `clients → Route 53 → CloudFront → WAF → ALB`
- HTTP: `ALB → web-frontend (port 3000), ALB → api-gateway (port 8080)`
- HTTP-REST: `api-gateway → auth-be:8081, prediagnostic-be:8000, message-producer:8082`
- HTTP-GRAPHQL: `web-frontend → api-gateway:8080/graphql`
- PostgreSQL: `auth-be → RDS PostgreSQL:5432`
- MongoDB: `prediagnostic-be → DocumentDB:27017`
- AMQP: `message-producer → Amazon MQ:5672 → notification-be`
- SMTP/SES: `notification-be → Amazon SES:587`
- S3 API: `auth-be → S3 (profile-images), prediagnostic-be → S3 (radiography-images)`
- Service Discovery: `api-gateway → AWS Cloud Map → backend services`

#### **🏛️ Description of Architectural Styles and Patterns Used:**

- **Client-Server**: Web browsers and mobile apps act as clients connecting through ALB
- **Microservices Architecture**: Independent, deployable services (auth-be, prediagnostic-be, notification-be, message-producer) with separate data stores
- **API Gateway Pattern**: Unified entry point for backend services with request composition and orchestration
- **Service Mesh (via AWS Cloud Map)**: Service discovery and dynamic routing between microservices
- **Load Balancer Pattern**: Application Load Balancer distributes traffic across multiple ECS tasks
- **Reverse Proxy Pattern**: ALB acts as reverse proxy with SSL/TLS termination and request filtering
- **Broker Pattern (Mediated Messaging)**: Amazon MQ decouples message producers from consumers
- **CQRS (Command Query Responsibility Segregation)**: Read operations to DocumentDB replicas, write operations to primary
- **Event-Driven Architecture**: Asynchronous notifications via message broker
- **Serverless Containers**: AWS Fargate eliminates server management
- **Database per Service**: Each microservice has its own database (RDS for auth, DocumentDB for prediagnostic)
- **Externalized Configuration**: Secrets Manager for credentials, Parameter Store for configuration
- **Health Check Pattern**: Multi-level health monitoring (ALB, ECS, application)
- **Circuit Breaker Pattern**: Prevent cascading failures between services
- **Bulkhead Pattern**: Service isolation through separate ECS services, databases, and resource limits

---

### 🚀 **Deployment Structure**

#### 🌐 **Deployment View**
*AWS cloud infrastructure and deployment configuration*

<div align="center">

![Deployment View](./images/deployment.png)

</div>

#### **🎯 Description of Architectural Elements and Relations:**

This view describes how the system is deployed on AWS infrastructure across multiple Availability Zones for high availability and fault tolerance.

**AWS Region and Availability Zones:**
- **Region**: `us-east-1` (N. Virginia)
- **Availability Zones**: `us-east-1a`, `us-east-1b`, `us-east-1c`
- **Multi-AZ Strategy**: All critical components deployed across at least 2 AZs

**Network Architecture (VPC):**
- **VPC CIDR**: `10.0.0.0/16`
- **Public Subnets** (3): Host ALB and NAT Gateways
  - `10.0.1.0/24` (us-east-1a), `10.0.2.0/24` (us-east-1b), `10.0.3.0/24` (us-east-1c)
- **Private Subnets** (3): Host ECS tasks
  - `10.0.11.0/24` (us-east-1a), `10.0.12.0/24` (us-east-1b), `10.0.13.0/24` (us-east-1c)
- **Database Subnets** (3): Host RDS and DocumentDB
  - `10.0.21.0/24` (us-east-1a), `10.0.22.0/24` (us-east-1b), `10.0.23.0/24` (us-east-1c)
- **Internet Gateway**: Provides internet access to public subnets
- **NAT Gateways** (3): One per AZ for private subnet internet access
- **VPC Endpoints**: Private connectivity to S3 and ECR

**Compute Layer (ECS Fargate):**
- **ECS Cluster**: `neumodiagnostics-cluster`
- **Container Orchestration**: AWS Fargate (serverless)
- **Service Distribution**: Tasks distributed across 3 AZs
- **Container Registry**: Amazon ECR (private repositories per service)

**Load Balancing:**
- **Application Load Balancer**: Internet-facing, spans 3 public subnets
- **Target Groups**: web-frontend-tg, api-gateway-tg
- **Health Checks**: HTTP checks every 30 seconds
- **SSL/TLS**: AWS Certificate Manager (ACM) certificates

**Database Layer:**
- **RDS PostgreSQL**: Multi-AZ deployment
  - Primary in us-east-1a, Standby in us-east-1b
  - Automatic failover in < 2 minutes
- **DocumentDB Cluster**: 3-node cluster
  - 1 writer + 2 readers across 3 AZs
  - Automatic failover in < 30 seconds

**Message Broker:**
- **Amazon MQ**: RabbitMQ cluster deployment
  - Active broker in us-east-1a, Standby in us-east-1b
  - Shared EBS storage for message persistence

**Storage:**
- **Amazon S3**: Region-scoped, automatically replicated across AZs
- **EBS Volumes**: For Amazon MQ persistent storage

**Service Dependencies:**
- ECS tasks depend on: RDS, DocumentDB, Amazon MQ availability
- Application Load Balancer depends on: At least one healthy ECS task per service
- NAT Gateways depend on: Internet Gateway
- All services depend on: AWS Secrets Manager for credentials

**Deployment Process:**
1. Infrastructure provisioned via Terraform/CloudFormation
2. Docker images built and pushed to ECR
3. ECS task definitions created with container specifications
4. ECS services created with auto-scaling policies
5. ALB target groups configured with health checks
6. DNS records created in Route 53
7. SSL certificates provisioned via ACM

#### **🏛️ Description of Architectural Patterns Used:**

- **Multi-AZ Deployment**: High availability across multiple Availability Zones
- **Immutable Infrastructure**: Containers replaced rather than updated
- **Infrastructure as Code**: Terraform/CloudFormation for reproducible deployments
- **Blue/Green Deployment**: Zero-downtime deployments via ECS
- **Auto-Scaling**: Dynamic resource adjustment based on demand
- **Service Mesh**: AWS Cloud Map for service discovery
- **Secrets Management**: Externalized credentials in Secrets Manager
- **Monitoring as Code**: CloudWatch alarms and dashboards defined in IaC

---

### 📚 **Layered Structure**

#### 🎂 **Layered View**
*Seven-tier layered architecture with AWS services*

<div align="center">

![Layered Structure](./images/layers.png)

</div>

#### **🎯 Description of Architectural Elements and Relations:**

Our system is structured in **seven distinct layers** (tiers), each with specific responsibilities:

**Layer 1: Presentation**
- **Purpose**: User interface and interaction
- **Components**: 
  - Web Front-end (Next.js) - ECS Fargate
  - Mobile App (future)
  - External System Clients
- **Relations**: Sends requests to Layer 2 (Load Balancing)

**Layer 2: Load Balancing and Security Gateway**
- **Purpose**: Traffic distribution, SSL termination, security filtering
- **Components**:
  - Application Load Balancer (ALB)
  - AWS WAF (Web Application Firewall)
  - Route 53 (DNS)
  - CloudFront (CDN - optional)
- **Relations**: 
  - Receives requests from Layer 1
  - Distributes to Layer 3 (API Gateway)
  - Enforces security policies

**Layer 3: Synchronous Orchestration**
- **Purpose**: Request routing, composition, authentication
- **Components**:
  - API Gateway [3-9 instances] (Go) - ECS Fargate
  - AWS Cloud Map (Service Discovery)
- **Relations**:
  - Receives requests from Layer 2
  - Routes to Layer 4 (Logic services)
  - Queries service registry for backend locations

**Layer 4: Logic (Business Services)**
- **Purpose**: Core business logic and functionality
- **Components**:
  - auth-be (Go) - ECS Fargate
  - prediagnostic-be (Python) - ECS Fargate
  - message-producer (Go) - ECS Fargate
  - notification-be (Go) - ECS Fargate
- **Relations**:
  - Processes requests from Layer 3
  - Accesses Layer 6 (Data) exclusively
  - Publishes events to Layer 5 (Asynchronous Communication)

**Layer 5: Asynchronous Communication**
- **Purpose**: Non-blocking message handling
- **Components**:
  - Amazon MQ (RabbitMQ) - Multi-AZ cluster
- **Relations**:
  - Receives messages from message-producer (Layer 4)
  - Delivers messages to notification-be (Layer 4)
  - Enables decoupled communication

**Layer 6: Data**
- **Purpose**: Data persistence and retrieval
- **Components**:
  - RDS PostgreSQL (Multi-AZ) - auth data
  - DocumentDB (Cluster) - clinical data
  - Amazon S3 - images and files
    - neumo-radiography-images
    - neumo-profile-images
    - neumo-backup-data
- **Relations**:
  - Accessed exclusively by Layer 4 services
  - Provides ACID transactions (RDS, DocumentDB)
  - Provides object storage (S3)

**Layer 7: External Communication**
- **Purpose**: Integration with external services
- **Components**:
  - Amazon SES (Email service)
  - External Healthcare Systems (via API)
  - Third-party Services
- **Relations**:
  - Invoked by Layer 4 (notification-be)
  - Extends system capabilities

**Layer Communication Rules:**
- Each layer can only communicate with adjacent layers (strict layering)
- Upper layers depend on lower layers
- Lower layers are unaware of upper layers
- Cross-layer communication prohibited (enforced by security groups)

#### **🏛️ Description of Architectural Patterns Used:**

- **7-Tier Layered Pattern**: Strict hierarchical organization
- **API Gateway Pattern**: Layer 3 orchestrates backend services
- **Load Balancer Pattern**: Layer 2 distributes traffic
- **Broker Pattern**: Layer 5 enables asynchronous messaging
- **Repository Pattern**: Layer 6 abstracts data access
- **Service Layer Pattern**: Layer 4 encapsulates business logic

**AWS Service Mapping:**
- **Layer 2**: AWS managed services (ALB, WAF, Route 53)
- **Layers 3-4**: ECS Fargate (serverless containers)
- **Layer 5**: Amazon MQ (managed message broker)
- **Layer 6**: Managed databases (RDS, DocumentDB) + S3
- **Layer 7**: AWS SES + external APIs

---

### 🧩 **Decomposition Structure**

#### 🔍 **Decomposition View**
*System breakdown into modules and functionalities*

<div align="center">

![Decomposition Structure](./images/decomposition.png)

</div>

#### **🎯 Description of Architectural Elements and Relations:**

This view decomposes the system into implementation units (modules and submodules) showing "is part of" relationships.

**Authentication Module** (implemented in `auth-be` ECS service)
- **Session Management** (submodule)
  - F1: Sign in
  - F2: Sign out
- **User Management** (submodule)
  - F3: Register user
  - F4: Upload profile picture

**Cases Management Module** (implemented in `prediagnostic-be` ECS service)
- **Query Management** (submodule)
  - F5: List pending cases
  - F6: List cases by patient
  - F7: List case by ID
- **Diagnostic Management** (submodule)
  - F8: Register medical diagnosis

**Prediagnostic Management Module** (implemented in `prediagnostic-be` ECS service)
- **Radiograph Management** (submodule)
  - F9: Upload radiograph
- **Prediagnostic Registration** (submodule)
  - F10: Register prediagnostic (AI prediction)

**Notifications Management Module** (implemented in `notification-be` ECS service)
- F11: Send email notifications

**Module-to-Service Mapping:**
- Authentication Module → `auth-be` ECS service → RDS PostgreSQL + S3 (profile images)
- Cases Management Module → `prediagnostic-be` ECS service → DocumentDB
- Prediagnostic Management Module → `prediagnostic-be` ECS service → DocumentDB + S3 (radiography images)
- Notifications Management Module → `notification-be` ECS service → Amazon SES

**Rationale for Module Organization:**
- Modules grouped by domain responsibility (authentication, diagnostics, notifications)
- Each module maps to one or more microservices
- Submodules represent cohesive functionality groups
- Facilitates parallel development and independent deployment

---

## 🎯 **Quality Attributes**

### **Security**

All security scenarios from Prototype 3 are maintained and enhanced:


## 🎯 **Quality Attributes**

### 🔒 **Security**

#### **Security Scenarios**

Our system implements four critical security scenarios to ensure data protection, user authentication, and secure communications:

##### **Scenario 1: Network Segmentation**

<div align="center">

![Network Segmentation Scenario](./images/Escenario%20-%20Network%20Segmentation%20Pattern.png)

</div>

**Description:**
- **Source (Fuente):** Person using their own computer
- **Stimulus (Estímulo):** Direct request sent to some component of the private network (Back-end and databases)
- **Artifact (Artefacto):** Private network components (Back-end and databases)
- **Environment (Ambiente):** System during its normal execution
- **Response (Respuesta):** Request rejection
- **Response Measure (Medición de la respuesta):** Number of requests made to private network components that have been rejected

**Applied Pattern:** Network Segmentation Pattern

---

##### **Scenario 2: Reverse Proxy**

<div align="center">

![Reverse Proxy Scenario](./images/Escenario%20-%20Reverse%20Proxy.png)

</div>

**Description:**
- **Source (Fuente):** External attacker or poorly maintained client
- **Stimulus (Estímulo):** Multiple malicious requests attempting to access backend services and overload the API Gateway
- **Artifact (Artefacto):** NginX configured as the single entry point
- **Environment (Ambiente):** System during its normal execution
- **Response (Respuesta):** The reverse proxy intercepts and blocks unauthorized access, filters and detects each request
- **Response Measure (Medición de la respuesta):** The reverse proxy registers and rejects illegitimate access, maintains and protects healthy instances

**Applied Pattern:** Reverse Proxy Pattern

---

##### **Scenario 3: Token Authentication (JWT)**

<div align="center">

![Token Authentication Scenario](./images/Escenario%20-%20Token%20Authentication.png)

</div>

**Description:**
- **Source (Fuente):** Malicious user without valid identification in the system
- **Stimulus (Estímulo):** Attempt to use any system functionality different from login or register
- **Artifact (Artefacto):** Set of functionalities that require authentication and a specific role (doctor or patient)
- **Environment (Ambiente):** System during its normal execution
- **Response (Respuesta):** Rejection of the request and redirection of the user to the login service or to their corresponding dashboard in case a lack of a valid JWT token identifying the user is detected
- **Response Measure (Medición de la respuesta):** Number of requests rejected due to missing valid JWT token

**Improves from Prototype 3:** There was a big vulnerability related to this pattern because somebody could steal the JWT from any user and use it in any other PC. To correct this vulnerability, now our JWT stores the ip of the user whe he logs in, and, when the user wants to send a request, the authentication system verifies if the JWT's ip is the same as the ip that is sending the request. If it is not, ther user is redirected to the login service.

**Applied Pattern:** Token-Based Authentication (JWT)

---

##### **Scenario 4: Secure Channel (HTTPS/TLS)**

<div align="center">

![Secure Channel Scenario](./images/Escenario%20-%20Secure%20Channel.png)

</div>

**Description:**
- **Source (Fuente):** User with bad intentions
- **Stimulus (Estímulo):** Attempt to intercept, read, or modify information transmitted between client and server during normal system communication
- **Artifact (Artefacto):** Secure communication channel implemented with HTTPS/TLS between client and reverse proxy
- **Environment (Ambiente):** System during its normal operation
- **Response (Respuesta):** Protection of communication through TLS encryption and rejection of any interception or data manipulation attempts
- **Response Measure (Medición de la respuesta):** Interception attempts blocked and traffic completely encrypted

**Applied Pattern:** Secure Channel Pattern (HTTPS/TLS)

---

#### **Applied Architectural Tactics**

Our system implements multiple security tactics organized by their defensive objectives:

##### **Resist Attacks**

- **Authenticate Actor:** JWT-based authentication system validates user identity before granting access to protected resources. Implemented in `auth-be` service with token validation at the API Gateway level.

- **Authorize Actors:** Role-based authorization checks ensure users can only access functionalities appropriate to their roles (doctors vs. patients). Enforced through middleware in the API Gateway and backend services.

- **Limit Access:** Network segmentation isolates private components (backend services and databases) from direct external access. Only the API Gateway is exposed as the single entry point.

- **Limit Exposure:** The API Gateway pattern minimizes the attack surface by exposing only necessary endpoints and hiding internal service topology from external clients.

- **Encrypt Data:** TLS/HTTPS encryption protects all data in transit between clients and the reverse proxy, and between internal services when handling sensitive information.

- **Separate Entities:** Microservices architecture separates concerns into independent services (`auth-be`, `prediagnostic-be`, `notification-be`), limiting the blast radius of potential security breaches.

##### **React to Attacks**

- **Revoke Access:** System can redirect the user to the service login to make him generate a new JWT, if the current JWT is invalid

---

#### **Applied Architectural Patterns**

- **Network Segmentation Pattern:** Isolation of private network components from direct external access
- **Reverse Proxy Pattern:** NginX as single entry point for filtering, load balancing, and security enforcement
- **Token-Based Authentication Pattern:** JWT for secure session management and stateless authentication
- **Secure Channel Pattern:** HTTPS/TLS encryption for all client-server communications

---

### **Performance and Scalability**


#### **Performance Scenarios**

Our system implements performance and scalability scenarios to ensure optimal resource utilization and response times under varying load conditions:

### **Scenario 1: Load Balancer / Weighted Round-Robin**

<div align="center">

![Load Balancer Scenario](./images/Escenario%20-%20Load%20Balancer.png)

</div>

**📋 Description:**
- **Source (Fuente):** 300 users
- **Stimulus (Estímulo):** Sending 300 different requests in 1 second
- **Artifact (Artefacto):** System
- **Environment (Ambiente):** System during its normal execution
- **Response (Respuesta):** Distribution of requests among the 3 API Gateway instances according to the Weighted Round-Robin algorithm
- **Response Measure (Medición de la respuesta):** Number of requests handled by each API Gateway instance

**Applied Pattern:** Load Balancer

### **Scenario 2: Throttling**

<div align="center">

![Load Balancer Scenario](./images/throught.png)

</div>

**📋 Description:**
- **Source (Fuente):** One or several users or a botnet.
- **Stimulus (Estímulo):** Sending more than 20 requests per minute from the same user.
- **Artifact (Artefacto):** System
- **Environment (Ambiente):** System during its normal execution
- **Response (Respuesta):** Limit the number of requests per minute from the same source establishing a rate limit through an intermediary (nginx)

- **Response Measure (Medición de la respuesta):** The number of requests accepted and rejected from the implied service.

**Applied Pattern:** **Throttling***: This pattern is used to limit access to some important resource or service. We can gracefully handle variations in demand.

We implemented this pattern through ***nginx*** setting the rate limit in the services we wanted. Next we can see an example about the implementation of this pattern. We are going to set a rate limit of 10 requests per minute with a burst of 1 to the register service exposed by the auth-be component. This means that our system will admit just 1 request each 6 seconds.

In order to make this scenario, we are going to do 5 requests in one second through *JMeter* and then we are going to see the logs.

<div align="center">

![Load Balancer Scenario](./images/pattern2Applied.png)
</div>


As we can see, only 2 of 5 requests were accepted. The reason to accept 2 of 5 is that we set a burst of 1, this means we have one more "emergency" request.

---

#### **Applied Architectural Tactics**

Our system implements performance tactics to optimize resource utilization and response times:

**Control Resource Demand**

- **Manage Work Requests:** The system processes incoming requests efficiently through the load balancer, distributing workload across multiple instances.

**Manage Resources**

- **Increase Resources:** Multiple API Gateway instances (3 instances) are deployed to handle increased load and provide redundancy.
- **Introduce Concurrency:** The Weighted Round-Robin algorithm distributes requests across multiple instances, enabling parallel processing of requests.
- **Maintain Multiple Copies of Computations:** Three instances of the API Gateway run simultaneously to handle concurrent requests without blocking.
- **Schedule Resources:** Weighted Round-Robin scheduling algorithm manages how requests are distributed among available API Gateway instances based on their weights and current load.

---

#### **Applied Architectural Patterns**

- **Load Balancer Pattern:** Weighted Round-Robin algorithm distributes incoming requests across multiple API Gateway instances

---

## **Reliability**

Our architecture implements multiple reliability patterns to ensure high availability, resilience, and fault tolerance across all system components.


### **Reliability Scenarios**

##### **Scenario 1: Replication Pattern (Hot Spare)**

<div align="center">

![Replication Pattern](./images/Escenario%20-%20Replication%20Pattern.png)

</div>

**Description:**
- **Source (Fuente):** User (doctor or patient)
- **Stimulus (Estímulo):** User sends a request which implies to make an action on the database auth-db
- **Artifact (Artefacto):** The system
- **Environment (Ambiente):** System with a failure on the database auth-db
- **Response (Respuesta):** The hot spare database is used to keep the system working
- **Response Measure (Medición de la respuesta):** Number of failed requests sent to the database 

**Applied Pattern:** Replication Pattern (Hot Spare)

##### **Scenario 2: Cluster Pattern**

<div align="center">

![CLuster Pattern](./images/Escenario%20-%20Cluster%20Pattern.png)

</div>

**Description:**
- **Source (Fuente):** Patient using the system from his PC
- **Stimulus (Estímulo):** User enters to his dashboard to se his radiographies and their states
- **Artifact (Artefacto):** The system
- **Environment (Ambiente):** System during the normal execution
- **Response (Respuesta):** The MongoDB cluster filters the radipgraphies by user's id and returns the information to visualize it in the front end
- **Response Measure (Medición de la respuesta):** Percentage of correctly returned cases

**Applied Pattern:** 

##### **Scenario 3: Load Balancer with Removal From Service Tactic implementend**

<div align="center">

![Load Balancer (Removal From Service)](./images/Escenario%20-%20Load%20Balancer%20(Removal%20from%20Service).png)

</div>

**Description:**
- **Source (Fuente):** User (doctor or patient)
- **Stimulus (Estímulo):** User sends a request
- **Artifact (Artefacto):** The system
- **Environment (Ambiente):** System with failes in one of the API Gateway instances
- **Response (Respuesta):**  The load balancer detects the failures in one of the instances and discards it for the next requests
- **Response Measure (Medición de la respuesta):** Number of requests without processing due to the failure in the instance of the API Gateway

**Applied Pattern:** Load Balancer

##### **Scenario 4: Service Discovery Pattern**

<div align="center">

![Service Discovery Pattern](./images/Escenario%20-%20Service%20Discovery%20Pattern.png)

</div>

**Description:**
- **Source (Fuente):** User (doctor or patient)
- **Stimulus (Estímulo):** User sends a request
- **Artifact (Artefacto):** The system
- **Environment (Ambiente):** The system during normal execution
- **Response (Respuesta):** The system detects all the available API Gateway instances to allow the load balancer to choose the instance that is going to process the request
- **Response Measure (Medición de la respuesta):** Number of requests that each instance has processed

**Applied Pattern:** Service Discovery Pattern

#### **Applied Architectural Tactics**

- **Redundant Spare**
- **Removal from service**

### **Interoperability Scenario**

##### **Scenario 1: Interoperability**

<div align="center">

![Interoperability](./images/Escenario%20-%20Interoperabilidad.png)

</div>

**Description:**
- **Source (Fuente):** Doctor using web-front-end
- **Stimulus (Estímulo):** The doctor makes a diagnostic for one of the cases
- **Artifact (Artefacto):**  The system
- **Environment (Ambiente):**  System during normal execution
- **Response (Respuesta):** The API GAteway coordinates the communication between different components (auth-be, prediagnostic-be, message_producer, etc.) to generate a response to the user
- **Response Measure (Medición de la respuesta):** Percentaje of requests fulfilled