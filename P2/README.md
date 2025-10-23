# 🚀 Delivery: Prototype 2
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

## Neuomodiagnostics
<div align="center">

![Team Logo](./images/logo.PNG)

</div>
---

## 🩺 Software System: **NeumoDiagnostics**

### 📋 Overview
**NeumoDiagnostics** is an AI-powered support platform designed to assist doctors in reviewing patient radiographs for pneumonia detection. Our system integrates advanced machine learning with comprehensive patient management features.

> ⚠️ **Important Note**: This model is designed to support, not replace, medical judgment. The final diagnosis always remains with the healthcare professional.

---

---

## 🏗️ **Architectural Structures**

Our NeumoDiagnostics system employs multiple architectural views to ensure comprehensive documentation and understanding of the system's design. Each view provides unique insights into different aspects of the architecture.

---

### 🔗 **Component and Connector (C&C) Structure**



#### 📊 **C&C View**
*Visual representation of system components and their interconnections*

<div align="center">

![C&C View](./images/cycview.png)

</div>

**🎯 Description of Architectural Elements and Relations:**
- Component interactions and communication patterns
- Connector types and protocols used
- Data flow between system components

**🏛️ Description of Architectural Styles and Patterns Used:**
- Architectural patterns implemented in the system
- Design decisions and their rationale

---

### 🚀 **Deployment Structure**



#### 🌐 **Deployment View**
*Infrastructure and deployment configuration overview*


**🎯 Description of Architectural Elements and Relations:**
- Hardware and software environment mapping
- Network topology and communication paths
- Resource allocation and distribution

**🏛️ Description of Architectural Patterns Used:**
- Deployment strategies and patterns
- Scalability and availability considerations

---

### 📚 **Layered Structure**

<div align="center">

![Layered Structure](./images/layers.png)

</div>

#### 🎂 **Layered View**
Next you can look the layered view, we recommend you to make zoom to each one of the layers to view what components belong to each one and view the logic of each component.


**🎯 Description of Architectural Elements and Relations:**

Our NeumoDiagnostics system is structured in **six distinct layers**, each with specific responsibilities and well-defined interactions:

---

### 🖼️ **Layer 1: Presentation**
- **Purpose**: User interface and interaction management
- **Components**: 
  - 🌐 Web Front-end
  - 💻 CLI Front-end
- **Relations**: Generates requests that are forwarded to the Synchronous Communication layer

---

### 🔄 **Layer 2: Synchronous Communication**
- **Purpose**: Real-time request routing and handling
- **Key Component**: 🚪 API Gateway
- **Relations**: 
  - Receives requests from Presentation layer
  - Routes requests to appropriate Logic layer components
  - Ensures synchronous communication patterns

---

### ⚙️ **Layer 3: Logic**
- **Purpose**: Core business logic and system functionality
- **Components**: 
    - prediagnostic-be
    - message-producer
    - notifications-be
    - auth-be
- **Relations**: 
  - Processes requests from API Gateway
  - Exclusive access to system data
  - Implements main system functionalities

---

### 📨 **Layer 4: Asynchronous Communication**
- **Purpose**: Non-blocking message handling
- **Technology**: 🐰 RabbitMQ (Message Broker)
- **Relations**: 
  - Manages asynchronous message queues
  - Enables system to continue processing while messages are queued
  - Supports decoupled component communication

---

### 💾 **Layer 5: Data**
- **Purpose**: Data storage and integrity management
- **Components**: 
  - prediagnostic-db
  - radiography-image-storage
  - users-db
  - profile-image-storage
- **Relations**: Provides persistent storage for all system data

---

### 🌐 **Layer 6: External Communication**
- **Purpose**: Integration with external services
- **Services**: 📧 Mailgun (Email API Platform)
- **Relations**: 
  - Extends system capabilities through external APIs
  - Handles communication with third-party services
  - Enables email notifications and external integrations


**🏛️ Description of Architectural Patterns Used:**
- Layered architecture implementation
- Separation of concerns principles

---

### 🧩 **Decomposition Structure**


#### 🔍 **Decomposition View**
*System breakdown into modules and their relationships*

<div align="center">

![Decomposition Structure](./images/decomposition.png)

</div>

**🎯 Description of Architectural Elements and Relations:**
- This view decomposes the system into implementation units (modules and submodules) and shows a pure “is part of” hierarchy. Each module encapsulates a specific set of functionalities, and submodules represent finer components within those modules.

- Modules and submodules (from the diagram) with their functionalities and implementation mapping to the repository under `Desarrollo/`:

	- Authentication module — implemented in `auth-be`
		- Session Management (is part of Authentication)
			- F1: Sign in
			- F2: Sign out
		- User Management (is part of Authentication)
			- F3: Register user
			- F4: Upload profile picture

	- Cases Management module — implemented in `prediagnostic-be` 
		- Query Management (is part of Cases Management)
			- F5: List pending cases
			- F6: List cases by patient
			- F7: List case by ID
		- Diagnostic Management (is part of Cases Management)
			- F8: Register medical diagnosis

	- Prediagnostic Management module — implemented in `prediagnostic-be`
		- Radiograph Management (is part of Prediagnostic Management)
			- F9: Upload radiograph
		- Prediagnostic Registration (is part of Prediagnostic Management)
			- F10: Register prediagnostic

	- Notifications Management module — implemented in `notification-be`
		- F11: Send notifications

- Relations: each submodule has a single parent (the module it belongs to) and encapsulates specific functionalities (F1 – F11). They're organized hierarchically to reflect their containment relationships. These functionalities are grouped according to their logical association within the system. 

- Intended uses of this view:
	- Communicate the functional structure to newcomers in digestible chunks (modules → submodules → functionalities).
	- Provide input for work assignment by module boundaries.
	- Reason about the impact and localization of changes (tree structure enables targeting the affected module/submodule without cross-module edits).
