# Delivery: Prototype 1
Software Arquitecture

Universidad Nacional de Colombia

## Team
* 1B
* Members:
  * Edinson Sanchez Fuentes - edsanchezf@unal.edu.co 
  * Adrian Ramirez Gonzalez - adramirez@unal.edu.co
  * Sergio Nicolas Siabatto Cleves - ssiabatto@unal.edu.co
  * Martin Polanco Barrero - mpolancob@unal.edu.co 
  * David Fernando Adames Rondon - dadames@unal.edu.co
  * Julian Esteban Mendoza Wilches - jmendozaw@unal.edu.co

## Software System
* Name: ***NeumoDiagnostics***
* Logo:
* Description: *Neumodiagnostics aims to serve as a support platform for doctors in the process of reviewing patient radiographs that may indicate pneumonia in the lungs. To this end, we have developed an AI model capable of predicting the presence of pneumonia in a radiograph. We have integrated this model into a comprehensive system that offers several functionalities for both patients and doctors. Before describing some of these functionalities, we must make one clarification: the model is not intended to replace medical judgment—the doctor has the final say in any diagnosis. Neumodiagnostics allow people to:*

  - (Patient) Upload a radiograph in JPG format.

  - (Patient) View all patient records according to their status (validated/processed).

  - (Patient) Select a single record to view its details.

  - (Doctor) View a list of cases that have not yet been evaluated by a doctor.

  - (Doctor) Select a single case from the entire list.

  - (Doctor) Evaluate the selected case.

## Architectural structures
### Component and connector (C & C) structure
- C & C view: Next it is shown the component and connector view:

![Component and connector view](images/cycview.png)

- Description of architectural styles used: We are using several architectural styles at both the connector and component levels. First let´s check the first ones:

  -  ***Client & Server***: Let´s remember first what is this style about: It is an architectural style in which the system’s functions are divided into two main roles: the client, which requests and consumes services, and the server, which processes the requests and provides the resources or functionalities. 
  
     *The Client-Server style was adopted because it allows a clear separation of responsibilities between clients (user interfaces or consuming applications) and the server (business logic and data management). This facilitates scalability, system maintenance, and enables multiple clients to access the same services in a centralized and controlled way. Moreover, it is a widely adopted model that relies on standard protocols such as HTTP, ensuring interoperability.*. If we see at c & c view we can appreciate some client- server relations between the components, some of this relations are:

      - front-end (Client) ---- businesslogic (Server)
      - businesslogic (Client) ---- blogicdb (Server)
      - businesslogic (Client) ---- prediagnostic (Server)
      - prediagnostic (Client) ---- pdiagndb (Server)
  -  ***Service-Based Architecture***: This style defines a way to make software components reusable through service interfaces. Services use common interface standards and an architectural pattern so they can be quickly incorporated into new applications. Our architecture adopts this style, defining several services in the components, like register service, authentication service, upload image service

