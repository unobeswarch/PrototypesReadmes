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

## 🎯 **Quality Attributes - AWS Implementation**

### **Security**

All security scenarios from Prototype 3 are maintained and enhanced:

**✅ Network Segmentation Pattern**
- Implemented via VPC with public/private/database subnet separation
- Private subnets for all application services and databases
- No direct internet access to backend components

**✅ Reverse Proxy Pattern**
- AWS Application Load Balancer serves as reverse proxy
- AWS WAF provides additional protection
- Rate limiting implemented at WAF level

**✅ Token Authentication (JWT)**
- Unchanged from Prototype 3
- JWT validation at API Gateway level
- IAM roles for service-to-service authentication

**✅ Secure Channel (HTTPS/TLS)**
- ALB handles TLS termination
- ACM-managed SSL certificates
- TLS 1.2+ enforced

**Additional Security Features:**
- AWS Shield for DDoS protection
- GuardDuty for threat detection
- CloudTrail for audit logging
- Secrets Manager for credential rotation

### **Performance and Scalability**

**✅ Load Balancer / Weighted Round-Robin**
- ALB distributes traffic across multiple ECS tasks
- Health checks ensure traffic only to healthy instances
- Cross-zone load balancing enabled

**✅ Throttling**
- AWS WAF rate limiting: 2000 requests per 5 minutes per IP
- API Gateway throttling (optional): 1000 requests/second
- DDoS protection via AWS Shield

**✅ Auto-Scaling**
- Horizontal scaling for all ECS services
- CPU and memory-based scaling policies
- Request count-based scaling
- Scheduled scaling for predictable patterns

**Enhanced Performance Features:**
- Multi-AZ deployment for high availability
- ECS Service Auto Scaling
- RDS Read Replicas for read-heavy operations
- DocumentDB read replicas across AZs
- CloudFront CDN for static content delivery

---

## 🔄 **Reliability (High Availability, Resilience, and Fault Tolerance)**

Our architecture implements multiple reliability patterns to ensure high availability, resilience, and fault tolerance across all system components.

---

### **Reliability Scenarios**

#### **Scenario 1: Replication Pattern (Hot Spare) - Database High Availability**

<!-- Diagram: Replication Pattern with RDS Multi-AZ and DocumentDB Cluster -->

**📋 Description:**
- **Source (Fuente):** Primary database instance (RDS or DocumentDB)
- **Stimulus (Estímulo):** Hardware failure, software crash, or AZ outage affecting the primary database
- **Artifact (Artefacto):** Database tier (RDS PostgreSQL Multi-AZ and DocumentDB Cluster)
- **Environment (Ambiente):** Production system under normal or high load
- **Response (Respuesta):** Automatic failover to standby replica with minimal service interruption
- **Response Measure (Medición de la respuesta):** 
  - Recovery Time Objective (RTO): < 2 minutes for RDS, < 30 seconds for DocumentDB
  - Recovery Point Objective (RPO): Zero data loss (synchronous replication)
  - Service availability maintained at > 99.9%

**Applied Pattern:** **Hot Spare Replication Pattern**

**Implementation:**

**RDS PostgreSQL (auth-db):**
- **Primary Instance**: Located in `us-east-1a`
- **Standby Replica**: Automatically provisioned in `us-east-1b` (synchronous replication)
- **Automatic Failover**: AWS RDS manages failover automatically
- **Endpoint**: Single DNS endpoint that automatically points to active primary
- **Data Replication**: Synchronous replication ensures zero data loss

**DocumentDB (prediagnostic-db):**
- **Primary Instance**: Writer instance in `us-east-1a`
- **Replica 1**: Reader instance in `us-east-1b`
- **Replica 2**: Reader instance in `us-east-1c`
- **Automatic Failover**: If primary fails, one replica is promoted to primary (< 30 seconds)
- **Read Distribution**: Read queries distributed across all replicas for load balancing

---

#### **Scenario 2: Service Discovery Pattern - Dynamic Service Location**

<!-- Diagram: Service Discovery Pattern with AWS Cloud Map -->

**📋 Description:**
- **Source (Fuente):** New ECS task instance or API Gateway instance
- **Stimulus (Estímulo):** Service instance starts, stops, or becomes unhealthy; auto-scaling adds/removes instances
- **Artifact (Artefacto):** ECS services and Application Load Balancer with target groups
- **Environment (Ambiente):** Dynamic cloud environment with auto-scaling
- **Response (Respuesta):** Services automatically discover and connect to healthy backend instances without manual configuration
- **Response Measure (Medición de la respuesta):**
  - Time to register new service: < 30 seconds
  - Time to deregister unhealthy service: < 60 seconds (2 failed health checks)
  - Zero manual DNS or configuration updates required

**Applied Pattern:** **Service Discovery Pattern**

**Implementation:**

**AWS Cloud Map (ECS Service Discovery):**
- **Namespace**: `neumodiagnostics.local` (private DNS namespace)
- **Service Discovery**: Each ECS service automatically registers with AWS Cloud Map
- **DNS-based Discovery**: Services discover each other via DNS queries

**Service Registry Configuration:**

| **Service** | **Discovery Name** | **Health Check** | **TTL** |
|-------------|-------------------|------------------|---------|
| api-gateway | api-gateway.neumodiagnostics.local | HTTP /health (port 8080) | 10s |
| auth-be | auth-be.neumodiagnostics.local | HTTP /health (port 8081) | 10s |
| prediagnostic-be | prediagnostic-be.neumodiagnostics.local | HTTP /health (port 8000) | 10s |
| message-producer | message-producer.neumodiagnostics.local | HTTP /health (port 8082) | 10s |

**Application Load Balancer Target Group Discovery:**
- **Dynamic Registration**: ECS tasks automatically register with ALB target groups upon startup
- **Health Checks**: ALB performs continuous health checks (30-second intervals)
- **Automatic Deregistration**: Unhealthy targets removed after 2 consecutive failed checks (60 seconds)
- **Connection Draining**: 30-second deregistration delay for graceful shutdown

**How It Works:**
1. New ECS task starts → Automatically registers with AWS Cloud Map and ALB target group
2. API Gateway queries `auth-be.neumodiagnostics.local` → Gets list of healthy auth-be instances
3. Instance becomes unhealthy → ALB marks as unhealthy and stops routing traffic
4. Failed health checks persist → Instance deregistered from service discovery
5. Auto-scaling adds new instance → Automatically discovered and receives traffic within 30 seconds

---

#### **Scenario 3: Cluster Pattern - Coordinated Service Groups**

<!-- Diagram: Cluster Pattern with ECS, DocumentDB, and Amazon MQ Clusters -->

**📋 Description:**
- **Source (Fuente):** Multiple instances of the same service type
- **Stimulus (Estímulo):** High traffic load requiring multiple service instances; individual instance failure
- **Artifact (Artefacto):** ECS Cluster with multiple service replicas, DocumentDB Cluster, Amazon MQ Cluster
- **Environment (Ambiente):** Production environment with variable load
- **Response (Respuesta):** Work is distributed across cluster members; failed members are replaced automatically; cluster continues operation
- **Response Measure (Medición de la respuesta):**
  - Cluster maintains operation with loss of up to 33% of members
  - New member provisioning time: < 2 minutes
  - Load distribution efficiency: > 85% across healthy members
  - Zero service disruption during member replacement

**Applied Pattern:** **Cluster Pattern**

**Implementation:**

**1. ECS Service Cluster (Compute Tier):**

Each service runs as a cluster of tasks distributed across multiple AZs:

| **Service Cluster** | **Min Tasks** | **Desired Tasks** | **Max Tasks** | **Distribution** |
|---------------------|---------------|-------------------|---------------|------------------|
| api-gateway-cluster | 3 | 3 | 9 | 1+ per AZ |
| auth-be-cluster | 2 | 2 | 6 | 1+ per AZ |
| prediagnostic-be-cluster | 2 | 3 | 6 | 1+ per AZ |
| web-frontend-cluster | 2 | 3 | 6 | 1+ per AZ |
| message-producer-cluster | 2 | 2 | 4 | 1+ per AZ |
| notification-be-cluster | 2 | 2 | 4 | 1+ per AZ |

**Cluster Coordination:**
- **ECS Service Scheduler**: Ensures tasks are distributed across AZs
- **Load Balancer**: Distributes requests across all healthy cluster members
- **Auto-Scaling**: Adds/removes cluster members based on load
- **Health Monitoring**: Unhealthy members automatically replaced

**2. DocumentDB Cluster (Database Tier):**

- **Cluster Configuration**: 1 Writer + 2 Readers across 3 AZs
- **Read Distribution**: Read queries load-balanced across reader instances
- **Write Operations**: All writes to primary (writer) instance
- **Automatic Failover**: Reader promoted to writer if primary fails

**3. Amazon MQ Cluster (Message Broker Tier):**

- **Active/Standby Configuration**: One active broker, one standby in different AZ
- **Automatic Failover**: Standby promoted to active on failure
- **Shared Storage**: EBS multi-attach for message persistence

**Cluster Benefits:**
- ✅ **No Single Point of Failure**: Loss of one member doesn't affect service
- ✅ **Horizontal Scalability**: Add members to increase capacity
- ✅ **Load Distribution**: Work distributed evenly across members
- ✅ **Fault Isolation**: Failed member isolated without affecting others
- ✅ **Rolling Updates**: Update members one at a time without downtime

---

#### **Scenario 4: Health Check and Circuit Breaker Pattern - Proactive Failure Detection**

<!-- Diagram: Health Check and Circuit Breaker Pattern -->

**📋 Description:**
- **Source (Fuente):** Load balancer, container orchestrator, or monitoring system
- **Stimulus (Estímulo):** Service instance becomes slow, unresponsive, or returns errors
- **Artifact (Artefacto):** All ECS services with health check endpoints, ALB target groups
- **Environment (Ambiente):** Production environment under varying load conditions
- **Response (Respuesta):** Unhealthy instances detected and removed from rotation; traffic routed only to healthy instances; alerts triggered for investigation
- **Response Measure (Medición de la respuesta):**
  - Health check frequency: Every 30 seconds
  - Unhealthy threshold: 2 consecutive failures (60 seconds to mark unhealthy)
  - Healthy threshold: 2 consecutive successes (60 seconds to mark healthy)
  - Zero requests routed to unhealthy instances after detection

**Applied Pattern:** **Health Check Pattern + Circuit Breaker Pattern**

**Implementation:**

**1. Application Load Balancer Health Checks:**

| **Target Group** | **Health Check Path** | **Interval** | **Timeout** | **Healthy Threshold** | **Unhealthy Threshold** |
|------------------|----------------------|--------------|-------------|----------------------|-------------------------|
| web-frontend-tg | GET / | 30s | 5s | 2 | 2 |
| api-gateway-tg | GET /health | 30s | 5s | 2 | 2 |

**Health Check Response Format (Standard for all services):**

```json
{
  "status": "healthy",
  "timestamp": "2025-12-08T10:30:00Z",
  "service": "api-gateway",
  "version": "1.0.0",
  "dependencies": {
    "database": "connected",
    "cache": "connected",
    "message_broker": "connected"
  },
  "metrics": {
    "uptime_seconds": 86400,
    "active_connections": 42,
    "memory_usage_mb": 512
  }
}
```

**2. ECS Container Health Checks:**

Each ECS task definition includes container-level health checks:

```json
"healthCheck": {
  "command": ["CMD-SHELL", "curl -f http://localhost:8080/health || exit 1"],
  "interval": 30,
  "timeout": 5,
  "retries": 3,
  "startPeriod": 60
}
```

**3. Deep Health Checks (Application-Level):**

Each service implements comprehensive health checks:

- **Shallow Health Check** (`/health`): Returns 200 OK if service is running
- **Deep Health Check** (`/health/ready`): Validates all dependencies:
  - Database connectivity (connection pool status)
  - Message broker connectivity
  - Dependent service availability
  - Disk space availability
  - Memory pressure check

**4. Circuit Breaker Implementation:**

API Gateway implements circuit breaker for downstream services:

- **Closed State**: Normal operation, requests forwarded
- **Open State**: After X failures, stop forwarding requests (fail fast)
- **Half-Open State**: After timeout, try single request to test recovery

**Circuit Breaker Configuration:**

| **Downstream Service** | **Failure Threshold** | **Timeout** | **Success Threshold** |
|------------------------|----------------------|-------------|----------------------|
| auth-be | 5 failures in 10s | 30s | 2 successes |
| prediagnostic-be | 5 failures in 10s | 30s | 2 successes |
| message-producer | 3 failures in 10s | 20s | 2 successes |

**Health Check Flow:**

```
1. ALB sends GET /health every 30s
   ↓
2. Service responds with health status + dependency checks
   ↓
3. IF response 200 OK within 5s → Healthy
   IF timeout or non-200 → Unhealthy
   ↓
4. After 2 consecutive unhealthy checks (60s total)
   → Mark target as unhealthy
   → Stop routing traffic
   → Trigger CloudWatch alarm
   ↓
5. ECS detects unhealthy container
   → Stop and replace container
   ↓
6. New container starts
   → 60s grace period (startPeriod)
   → Begin health checks
   ↓
7. After 2 consecutive healthy checks (60s total)
   → Mark target as healthy
   → Resume routing traffic
```

---

### **Applied Reliability Tactics**

Our architecture implements multiple tactics from the reliability category:

#### **Detect Faults**

- **Ping/Echo (Health Checks)**: Regular health checks at ALB and ECS levels
- **Monitor**: CloudWatch metrics tracking service health, response times, and error rates
- **Heartbeat**: ECS tasks send regular status updates to container orchestrator
- **Exception Detection**: Application logs errors and exceptions to CloudWatch

#### **Recover from Faults**

- **Active Redundancy (Hot Spare)**: RDS standby replica, DocumentDB replicas, multiple ECS tasks
- **Retry**: Failed requests automatically retried with exponential backoff
- **Rollback**: Failed deployments automatically rolled back to previous version
- **Software Upgrade**: Rolling updates replace instances one at a time
- **Exception Handling**: Services implement graceful error handling and recovery

#### **Prevent Faults**

- **Removal from Service**: Unhealthy instances removed from load balancer rotation
- **Predictive Model**: Auto-scaling based on predicted traffic patterns
- **Increase Competence Set**: Multi-AZ deployment ensures redundancy across failure domains

---

## 🔗 **Interoperability**

Our architecture ensures seamless interoperability with external systems and services through well-defined interfaces and standard protocols.

---

### **Interoperability Scenario**

<!-- Diagram: Interoperability with External Systems via API Gateway -->

**📋 Description:**
- **Source (Fuente):** External healthcare systems, third-party medical imaging systems, email service providers, future mobile applications
- **Stimulus (Estímulo):** External system needs to integrate with NeumoDiagnostics to exchange patient data, retrieve diagnostic results, or send notifications
- **Artifact (Artefacto):** API Gateway with RESTful and GraphQL interfaces, Amazon SES integration, S3 API for image storage
- **Environment (Ambiente):** Integration requests from various external systems with different protocols and data formats
- **Response (Respuesta):** System provides standardized APIs supporting multiple protocols (REST, GraphQL, HTTPS), handles authentication/authorization, transforms data formats, and maintains backward compatibility
- **Response Measure (Medición de la respuesta):**
  - API response time for external requests: < 500ms (p95)
  - API availability for external integrations: > 99.9%
  - Support for multiple data formats: JSON, XML (via content negotiation)
  - API versioning support: Maintain backward compatibility for at least 2 major versions
  - Authentication success rate: > 99.5%

**Applied Pattern:** **API Gateway Pattern + Adapter Pattern**

---

### **Interoperability Implementation**

#### **1. RESTful API (External Integration)**

**Public API Endpoints:**

```
Base URL: https://api.neumodiagnostics.com/v1
```

| **Endpoint** | **Method** | **Purpose** | **Authentication** |
|--------------|------------|-------------|-------------------|
| `/auth/token` | POST | Obtain JWT token for external systems | API Key |
| `/patients/{id}` | GET | Retrieve patient information | JWT |
| `/cases/{id}` | GET | Retrieve diagnostic case | JWT |
| `/radiographs` | POST | Upload radiograph image | JWT |
| `/diagnoses/{id}` | GET | Retrieve final diagnosis | JWT |
| `/health` | GET | Health check endpoint | None |

**API Request/Response Format (JSON):**

```json
// Request: GET /patients/12345
{
  "patient_id": "12345",
  "include": ["demographics", "recent_cases"]
}

// Response: 200 OK
{
  "patient_id": "12345",
  "demographics": {
    "name": "John Doe",
    "age": 45,
    "gender": "M"
  },
  "recent_cases": [
    {
      "case_id": "C-789",
      "date": "2025-12-01",
      "status": "diagnosed"
    }
  ]
}
```

**API Versioning Strategy:**

- **URL Versioning**: `/v1/`, `/v2/` in URL path
- **Header Versioning**: `Accept: application/vnd.neumodiagnostics.v1+json`
- **Deprecation Policy**: Version maintained for 12 months after new version release

#### **2. GraphQL API (Flexible Query Interface)**

**GraphQL Endpoint:**

```
https://api.neumodiagnostics.com/graphql
```

**Sample GraphQL Query:**

```graphql
query GetPatientWithCases($patientId: ID!) {
  patient(id: $patientId) {
    id
    name
    email
    cases {
      id
      createdAt
      status
      prediagnostic {
        prediction
        confidence
      }
      diagnosis {
        conclusion
        doctorName
      }
    }
  }
}
```

**Benefits for External Systems:**
- ✅ Query only required fields (avoid over-fetching)
- ✅ Single request for related data (reduce round trips)
- ✅ Strong typing and schema introspection
- ✅ Real-time updates via GraphQL subscriptions

#### **3. Email Service Integration (Amazon SES)**

**External Email Provider Interface:**

```
Protocol: SMTP / SES API
Endpoint: email-smtp.us-east-1.amazonaws.com
Port: 587 (TLS)
```

**Email Templates:**

| **Template** | **Trigger** | **Recipient** |
|--------------|-------------|---------------|
| case_assigned | New case assigned to doctor | Doctor |
| prediagnostic_ready | AI prediction completed | Doctor |
| diagnosis_ready | Final diagnosis available | Patient |
| account_verification | New user registration | User |

**Integration Points:**
- **Outbound**: NeumoDiagnostics → Amazon SES → External email addresses
- **Inbound**: Bounce/complaint handling via SNS webhooks

#### **4. Storage API Integration (Amazon S3)**

**Image Storage Interface:**

```
Protocol: HTTPS (S3 API)
Endpoint: s3.us-east-1.amazonaws.com
```

**Pre-Signed URL Generation (Secure External Upload):**

External systems can obtain temporary upload URLs:

```bash
POST /api/v1/radiographs/upload-url
Authorization: Bearer <JWT>

Response:
{
  "upload_url": "https://neumo-radiography-images.s3.amazonaws.com/...",
  "expires_in": 300,
  "max_size_mb": 50
}
```

**Benefits:**
- ✅ Direct upload to S3 (bypass application servers)
- ✅ Temporary credentials (expire after use)
- ✅ Reduced server load

#### **5. Authentication and Authorization (External Systems)**

**API Key Authentication (System-to-System):**

```bash
POST /auth/token
Content-Type: application/json
X-API-Key: <external-system-api-key>

{
  "system_id": "external-hospital-A",
  "scopes": ["read:patients", "read:cases"]
}

Response:
{
  "access_token": "eyJhbGc...",
  "token_type": "Bearer",
  "expires_in": 3600
}
```

**OAuth 2.0 Support (Future Enhancement):**

```
Authorization Endpoint: /oauth/authorize
Token Endpoint: /oauth/token
Supported Grant Types: authorization_code, client_credentials
```

#### **6. Data Format Transformation (Adapter Pattern)**

**Content Negotiation:**

```bash
# Request JSON response (default)
GET /patients/12345
Accept: application/json

# Request XML response
GET /patients/12345
Accept: application/xml
```

**FHIR Compatibility (Healthcare Standard):**

```bash
# FHIR-compliant patient resource
GET /fhir/Patient/12345
Accept: application/fhir+json

Response (FHIR R4 format):
{
  "resourceType": "Patient",
  "id": "12345",
  "name": [{
    "use": "official",
    "family": "Doe",
    "given": ["John"]
  }],
  "gender": "male",
  "birthDate": "1980-01-01"
}
```

---

### **Applied Interoperability Tactics**

#### **Locate**

- **Service Discovery**: AWS Cloud Map for internal service location
- **DNS**: Route 53 for external API endpoint resolution

#### **Manage Interfaces**

- **API Gateway**: Single entry point for all external requests
- **Versioning**: URL and header-based API versioning
- **Documentation**: OpenAPI/Swagger specification for API documentation

#### **Coordinate**

- **Orchestrate**: API Gateway orchestrates calls to multiple backend services
- **Event-Based**: Asynchronous notifications via message queue (Amazon MQ)

#### **Exchange Data**

- **Translate**: API Gateway transforms between external and internal data formats
- **Adapter Pattern**: Service adapters for each external integration (SES, S3)
- **Standard Protocols**: REST (JSON), GraphQL, HTTPS, SMTP

---

## 📊 **Prototype 4 Compliance Summary**

### **✅ Requirements Fulfilled**

#### **Functional Requirements**
- ✅ Complete functional coverage defined for NeumoDiagnostics medical platform
- ✅ All modules documented in Decomposition Structure (Authentication, Cases Management, Prediagnostic, Notifications)

#### **Non-Functional Requirements**

**Reliability Scenarios (4 Required):**
- ✅ **Scenario 1**: Replication Pattern (Hot Spare) - RDS Multi-AZ and DocumentDB Cluster
- ✅ **Scenario 2**: Service Discovery Pattern - AWS Cloud Map + ECS Service Discovery
- ✅ **Scenario 3**: Cluster Pattern - ECS Clusters, DocumentDB Cluster, Amazon MQ Cluster  
- ✅ **Scenario 4**: Health Check and Circuit Breaker Pattern (team-defined)

**Interoperability:**
- ✅ Interoperability scenario with external systems via RESTful API, GraphQL, Amazon SES

**Security (Maintained from Prototype 3):**
- ✅ Network Segmentation Pattern (VPC with public/private subnets)
- ✅ Reverse Proxy Pattern (Application Load Balancer + AWS WAF)
- ✅ Token Authentication (JWT)
- ✅ Secure Channel (HTTPS/TLS via ACM)

**Performance and Scalability (Maintained from Prototype 3):**
- ✅ Load Balancer Pattern (ALB with multi-AZ distribution)
- ✅ Throttling (AWS WAF rate limiting)
- ✅ Auto-Scaling (ECS Service Auto Scaling)

#### **Architectural Structures**
- ✅ **Component and Connector (C&C) Structure**: Updated for AWS deployment
- ✅ **Deployment Structure**: Complete AWS multi-AZ infrastructure
- ✅ **Layered Structure**: 7-tier architecture with AWS services
- ✅ **Decomposition Structure**: Module breakdown with service mapping

---

## 👥 **Team Contact**

For questions about this architecture or deployment assistance:

📧 **Technical Lead**: edsanchezf@unal.edu.co  
📧 **Architecture Team**: Team 1B  
🏛️ **Institution**: Universidad Nacional de Colombia

---

**Document Version**: 1.0  
**Last Updated**: December 8, 2025  
**Status**: Prototype 4 - AWS Deployment Architecture