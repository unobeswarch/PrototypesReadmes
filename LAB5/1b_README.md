# 🚀 Delivery: Laboratory 5 - Network Segmentation Pattern
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

## 📊 Network Segmentation Pattern Implementation

### Pattern Description
The Network Segmentation Pattern is a security architectural pattern that involves dividing a network into smaller segments or subnets to improve security and performance. This pattern helps control access between different parts of the system and limits the potential spread of security threats by creating isolated network zones for different types of components.

### Quality Scenario Addressed
**Security Focus**: Confidentiality and Access Control
- **Stimulus**: Potential unauthorized access attempts between different system components
- **Response**: Network isolation prevents direct access to sensitive internal services through subnet segmentation
- **Metric**: Complete isolation of internal services from public access, with controlled access only through designated entry points

### Implementation Steps

1. Network Definition and Segmentation

   **Private Network (192.168.10.0/26)**
   - Network Class: C
   - Valid IP Range: 192.168.10.1 - 192.168.10.62
   - Subnet Mask: 255.255.255.192
   - Components Segmentation:
   * Backend Segment (192.168.10.1 - 192.168.10.30)
   * Data Segment (192.168.10.31 - 192.168.10.50)
   * Frontend Segment (192.168.10.51 - 192.168.10.55)
   * Others Segment (192.168.10.56 - 192.168.10.62)

   **Public Network (172.30.0.0/24)**
   - Network Class: B
   - Valid IP Range: 172.30.0.1 - 172.30.0.254
   - Subnet Mask: 255.255.255.0

2. Logical Service Organization

   Backend Services Zone:
   ```
   Services in private network (192.168.10.1-30):
   - API Gateway
   - Prediagnostic Backend
   - Authentication Backend
   - Message Producer
   - Message Broker
   - Notification Backend
   ```

   Data Services Zone:
   ```
   Services in private network (192.168.10.31-50):
   - Prediagnostic Database
   - Authentication Database
   ```

   Frontend Services Zone:
   ```
   Dual-network services:
   - Web Frontend
   * Private network access (192.168.10.0/26)
   * Public network access (172.30.0.0/24)
   - CLI Frontend
   * Private network access (192.168.10.0/26)
   * Public network access (172.30.0.0/24)
   ```

   Network Verification Results:
   ```
   Private Network (192.168.10.0/26):
   - All backend services successfully isolated
   - Databases properly segmented
   - Frontend services with internal access
   - Network marked as internal: true

   Public Network (172.30.0.0/24):
   - Only frontend services present
   - No direct access to backend services
   - Network marked as internal: false
   ```

### Architectural View

For this lab we used the **Component and Connector View** to represent/illustrate the different network subnets and what components are located in each subnet:

![Component and Connector View - Subnet Representation ](./images/1b_cycview.png)

#### Network Segmentation Details

1. **Public Network Access (172.30.0.0/24)**
   - Web and CLI frontends exposed to public network
   - Acts as entry point for user interactions

2. **Private Network Segmentation (192.168.10.0/26)**
   - Backend services isolated in dedicated segment
   - Data tier secured in separate segment
   - Frontend private interfaces in controlled segment
   - Reserved segment for future expansion

3. **Security Controls**
   - API Gateway as central access control point
   - Data segment accessible only by authorized backend services
   - Strict isolation between segments
   - Controlled communication paths

### Configuration Details

```yaml
networks:
  private:
    ipam:
      driver: default
      config:
        - subnet: 192.168.10.0/26    # Internal services network
    internal: true                    # Ensures network isolation from external access
    labels:
      description: "Backend and Data Services Network"

  public:
    ipam:
      driver: default
      config:
        - subnet: 172.30.0.0/24      # External access network
    labels:
      description: "Frontend Public Access Network"
```

This configuration:
- Defines clear network boundaries
- Implements network isolation through the `internal` flag
- Uses descriptive labels for better documentation
- Allows Docker to handle IP assignment dynamically
- Maintains security through network segmentation

### Results and Improvements

1. Enhanced Security through Segmentation:
   - Backend services (192.168.10.1-30): Isolated core services
   - Data tier (192.168.10.31-50): Protected database systems
   - Frontend segment (192.168.10.51-55): Controlled public access
   - Support services (192.168.10.56-62): Isolated maintenance zone

2. Multi-layered Security Benefits:
   - Physical network separation between public and private networks
   - Logical segmentation within the private network
   - Controlled inter-segment communication
   - Reduced attack surface through precise IP assignments

3. Access Control Improvements:
   - Only frontend services have dual-network presence
   - All internal services are completely isolated from public access
   - API Gateway serves as the single entry point for backend services
   - Databases are restricted to the data segment

### Recommendations

1. Network Security Monitoring:
   - Implement network flow monitoring between segments
   - Set up intrusion detection at segment boundaries
   - Regular security scanning of both public and private networks

2. Access Control Management:
   - Maintain strict firewall rules between segments
   - Regular audit of network access patterns
   - Document all inter-segment communication requirements

3. Security Maintenance:
   - Regular review of IP assignments within segments
   - Periodic validation of network isolation
   - Update segment boundaries based on new service requirements
   - Implement automated network security testing

4. Additional Security Measures:
   - Network encryption between segments
   - Strong authentication for cross-segment access
   - Detailed logging of all inter-segment communication
   - Regular security training for development team