# 🚀 Delivery: Prototype 1
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

---

## 🩺 Software System: **NeumoDiagnostics**

### 📋 Overview
**NeumoDiagnostics** is an AI-powered support platform designed to assist doctors in reviewing patient radiographs for pneumonia detection. Our system integrates advanced machine learning with comprehensive patient management features.

> ⚠️ **Important Note**: This model is designed to support, not replace, medical judgment. The final diagnosis always remains with the healthcare professional.

### ✨ Key Features

#### 👤 **For Patients:**
- 📤 Upload radiographs in JPG format
- 📊 View all medical records by status (validated/processed)
- 🔍 Access detailed record information

#### 👨‍⚕️ **For Doctors:**
- 📋 Review pending evaluation cases
- 🎯 Select and examine individual cases
- ✅ Evaluate and validate diagnoses

---

## 🏗️ Architectural Structures

### 🔗 Component and Connector (C & C) Structure

#### 📊 **Architecture Diagram**
The following diagram illustrates our component and connector view:

![Component and connector view](images/cycview.png)

- Description of architectural styles used: We are using several architectural styles at both the connector and component levels. First let´s check the first ones:

  -  ***Client & Server***: Let´s remember first what is this style about: It is an architectural style in which the system’s functions are divided into two main roles: the client, which requests and consumes services, and the server, which processes the requests and provides the resources or functionalities. 
<div style="background-color: #f6f8fa; padding: 15px; border-radius: 8px; margin: 10px 0; color: #333333;">

**🎯 Why we adopted this style:**

*The Client-Server style was adopted because it allows a clear separation of responsibilities between clients (user interfaces or consuming applications) and the server (business logic and data management). This facilitates scalability, system maintenance, and enables multiple clients to access the same services in a centralized and controlled way. Moreover, it is a widely adopted model that relies on standard protocols such as HTTP, ensuring interoperability.*

</div>

**🔗 Client-Server Relations in our C&C View:**

| 🖱️ **Client** | ↔️ | 🖥️ **Server** |
|---------------|---|---------------|
| `front-end` | ➡️ | `businesslogic` |
| `businesslogic` | ➡️ | `blogicdb` |
| `businesslogic` | ➡️ | `prediagnostic` |
| `prediagnostic` | ➡️ | `pdiagndb` |

---

### 🔧 **Service-Based Architecture**

<div style="background-color: #fff8e1; padding: 15px; border-radius: 8px; border-left: 4px solid #ff9800; color: #333333;">

A service-based architecture was adopted since the main components of the system expose their functionality through reusable services with well-defined interfaces, enabling their consumption by other modules or applications. This approach provides **flexibility**, **scalability**, and **separation of concerns**, ensuring standardized and organized communication between the different components.

</div> 
<div style="background-color: #ffeaa7; padding: 10px; border-radius: 5px; margin: 10px 0; color: #333333;">

**⚠️ Important Note:** Although not all the characteristics of a microservices architecture are met (such as complete independence in deployment and persistence) for example, our BusinessLogic component exposes services through its interfaces, such as authentication, registration, and radiograph upload. Similarly, our PreDiagnostic component exposes services like pneumonia prediction, pre-diagnostic, and diagnostic. As we can see, although we are using a service-based approach, not all services are fully decoupled from the main component that holds them.

</div>

---

### 🌐 **REST Architectural Style**

<div style="background-color: #e8f5e8; padding: 15px; border-radius: 8px; border-left: 4px solid #28a745; color: #333333;">

The REST architectural style was implemented for services that demand a stable and predictable contract structure, as well as for internal communication where data over-fetching does not represent a critical concern.

</div>
#### 🔐 **Identity and Authorization Services**

<div style="background-color: #f8f9fa; padding: 10px; border-radius: 5px; margin: 10px 0; color: #333333;">

- **Implementation:** The implementation of REST is applied to the authorization (login) and registration services.

</div>

**📋 Justification:**

<div style="background-color: #fff3cd; padding: 12px; border-radius: 6px; border-left: 3px solid #ffc107; color: #333333;">

- **🔒 Fixed and Low Evolution Data Contract:**  
  The authentication and registration endpoints possess an inherently rigid data contract. The request and response formats for these transactional operations rarely change (e.g., username and password as input; session token as output).

- **🛡️ Simplicity and Robustness:**  
  REST, by offering a fixed structure and leveraging standard HTTP operations (POST, GET), ensures simple, robust integration with low coupling, which is ideal for these low-volatility services. The added complexity of a flexible querying engine like GraphQL is not justified when the required data is constant.

</div>
#### 🔄 **Inter-Component Communication**

<div style="background-color: #f8f9fa; padding: 10px; border-radius: 5px; margin: 10px 0; color: #333333;">

- **Implementation:** REST is utilized for internal communication between the Business Logic (BusinessLogic) component and the Prediagnostic (Prediagnostic) component.

</div>

**📋 Justification:**

<div style="background-color: #d1ecf1; padding: 12px; border-radius: 6px; border-left: 3px solid #17a2b8; color: #333333;">

- **📦 Need for Complete Data:**  
  The services defined in the Prediagnostic component require the transmission of complete and defined data sets for their correct operation. Flexibility in field selection is not a functional requirement within this internal processing context.

</div>

---

### ⚡ **GraphQL**

<div style="background-color: #f3e5f5; padding: 15px; border-radius: 8px; border-left: 4px solid #9c27b0; color: #333333;">

GraphQL was selected to manage communication between the Frontend (Client) and the Business Logic (BusinessLogic) component due to its ability to optimize data transfer and support variable information requirements.

</div>
**📋 Justification:**

#### 🚫 **Resolution of Over-fetching:**

<div style="background-color: #fce4ec; padding: 12px; border-radius: 6px; margin: 10px 0; color: #333333;">

The primary reason for adopting GraphQL is its ability to allow the client to request only the specific data fields it needs (e.g., just the case ID and status, without the entire information of the case).

This optimizes performance, reduces server load, and minimizes bandwidth consumption, which is crucial for the user experience in dashboards with dynamic visualization requirements.

</div>

#### 🔄 **Dual Architecture Approach:**

<div style="background-color: #e1f5fe; padding: 15px; border-radius: 8px; border-left: 4px solid #00bcd4; color: #333333;">

This dual architecture ensures that the most appropriate tool is used for each task:

- **🔧 REST Benefits:** Leveraging the simplicity of REST for stable transactions.
- **⚡ GraphQL Benefits:** Taking advantage of the flexibility of GraphQL for variable data retrieval and efficiency in the presentation layer.

</div>

---

<div align="center" style="background: linear-gradient(90deg, #667eea 0%, #764ba2 100%); padding: 20px; border-radius: 10px; color: white; margin: 20px 0;">

### 🏆 **NeumoDiagnostics**


</div>


