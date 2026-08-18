# Enterprise-Network-Segmentation
## ACL and Zone-Based Firewall Design in Cisco Packet Tracer
## Author: Chisomebi Anunobi Date: 8/17/2026 Tools Used: Cisco Packet Tracer V.9.0.1.08578 Cisco IOS V15.1

## 1 Overview 
In this project i simulated a small enterprise network with 3 security zones (HR, Employees, and Servers) and implemented 2 levels of access control: 
stateless packet filtering using ACLs (access command lists) and stateful packet filtering using Cisco's Zone based policy firewall (ZFW).
The goal of this project was to show how network segmentation and layered access control can be used to enforce an organizations security policy on the infrastructure level. 

## 2 Security Policy 
HR systems and Payroll servers contain sensitive employee data and so they must be inaccessible to the general employee network. 
Only the HR zone should be able to access the Payroll server to allow for distributing pay but the File server should be accessible from either the HR or Employee zones.

Access Policy Table 
<img width="623" height="168" alt="image" src="https://github.com/user-attachments/assets/f7ba19d1-9e27-4f3e-81bb-58117bcdc8d2" />

## 3 Network Topology

<img width="1362" height="571" alt="image" src="https://github.com/user-attachments/assets/39dd586e-58e9-44a3-ab4c-94880a73c8bf" />

IP Addressing Table 
<img width="605" height="238" alt="image" src="https://github.com/user-attachments/assets/e27f0de0-5a3d-4b8f-a779-8ca7ba869b0d" />

## 4 Implementation: Access Control Lists
### 4.1 Design rationale 
Both standard and extended rules are made, the standard rules can be used to completely isolate the  employee subnet from accessing the HR subnet, but in our extended list we also make rules that would lead to the same outcome so the standard ACL is not activated and instead we rely on the extended rules list. As ACL rules are based on first match principles , rules are organized most specific to least specific which leaves us with , "permit ip any any" as our last line. This is used to avoid the implicit deny default behavior as any packets not explicitly denied by our rules should be accepted. 

### 4.2 Configuration
these are the config queries ran : 
! Standard 
Router(config)#access-list 10 deny 192.168.20.0 0.0.0.255
Router(config)#access-list 10 permit any
! Extended 
Router(config)#access-list 100 permit ip 192.168.10.0 0.0.0.255 192.168.30.10 0.0.0.0
Router(config)#access-list 100 deny ip 192.168.20.0 0.0.0.255 host 192.168.30.2
Router(config)#access-list 100 permit ip 192.168.20.0 0.0.0.255 host 192.168.30.3
Router(config)#access-list 100 deny ip 192.168.20.0 0.0.0.255 192.168.10.0 0.0.0.255
Router(config)#access-list 100 permit ip any any

### 4.3 Verification
<img width="514" height="166" alt="image" src="https://github.com/user-attachments/assets/fe9c9d2f-f63a-4b14-bc62-9c00fc9f1bd6" />

### 4.4 Testing Results

<img width="443" height="133" alt="image" src="https://github.com/user-attachments/assets/913980d6-7efc-469f-b686-7da1ed835e1b" />
<img width="499" height="179" alt="image" src="https://github.com/user-attachments/assets/3514840f-ad3d-472d-9d44-acd379f5ea33" />
<img width="491" height="203" alt="image" src="https://github.com/user-attachments/assets/ca7faea5-30d2-41e2-8ebb-d8c7bd895d64" />
<img width="507" height="170" alt="image" src="https://github.com/user-attachments/assets/18e0cd06-23b0-48e5-8372-d48c328845e5" />

### 4.5 Issues Encountered and Resolved 

Issue : ACL showed 0 match count despite being given traffic that should have triggered earlier deny rules.

Cause: ACL was applied with "ip access-group 100 in " on the server facing interface meaning it was only active on packets coming into the router through the server interface, and not traffic going into the server subnet. 

Solution: Changed command to "ip access-group 100 out" on the server facing interface and the Hr facing interface. This change allowed packets to be filtered as they were leaving the router towards the HR and Server facing interfaces.

Lesson Learned: it is important to evaluate ACL direction depending on each  interface and protected zone to be reachable from a different interface needs its own specific application of the policy. 

## 5 Implementation: Zone-Based Policy Firewall (ZFW)

### 5.1 Design Rationale
ACLs are stateless packet filtering so return traffic requires a separate explicit rule as per the access list rules made above. 
ZFW uses stateful packet filtering meaning it can remember the history of the packet as well as connection it has already approved, which allows it to automatically approve return traffic for connections it has already validated, this method more closely resembles how a production firewall should behave.  

### 5.2  Prerequisite : enable Security License
<img width="652" height="100" alt="image" src="https://github.com/user-attachments/assets/bc51c95f-d54d-4524-8355-dfcbcb3498b1" />
My cisco packet tracer already has the license installed, otherwise run these commands : 

R1#configure terminal

R1(config)#license boot module c2900 technology-package securityk9
 > then accept EULA


R1(config)#exit

R1#copy running-config startup-config

R1#reload

### 5.3 Configuration

> Define the zones


zone security HR-ZONE

zone security SERVER-ZONE

zone security EMPLOYEE-ZONE

exit

> Define traffic classes (reusing ACL 100 as match criteria)


class-map type inspect match-any HR-TO-SERVER

match access-group 100

exit

>class-map type inspect match-any EMPLOYEE-TO-SERVER


match access-group 100

exit

> Define policies (what to do with matched traffic)


policy-map type inspect HR-POLICY

class type inspect HR-TO-SERVER

inspect

exit

>policy-map type inspect EMPLOYEE-POLICY


class type inspect EMPLOYEE-TO-SERVER

inspect

exit

> Apply policies between zone pairs


zone-pair security HR-TO-SERVER-PAIR source HR-ZONE destination SERVER-ZONE

service-policy type inspect HR-POLICY

exit

zone-pair security EMPLOYEE-TO-SERVER-PAIR source EMPLOYEE-ZONE destination SERVER-ZONE

service-policy type inspect EMPLOYEE-POLICY

exit

> Assign interfaces to zones


interface g0/0

zone-member security HR-ZONE

exit

interface g0/1

zone-member security EMPLOYEE-ZONE

exit


interface g0/2

zone-member security SERVER-ZONE

exit

### 5.4 Testing
<img width="476" height="263" alt="image" src="https://github.com/user-attachments/assets/f4ce89c8-0c26-4155-a917-44068c4f894a" />
HR to Payroll server 

<img width="492" height="177" alt="image" src="https://github.com/user-attachments/assets/f105bec4-fc64-4d41-87ab-dd8dcb8816b2" />
Employee to payroll server

### 5.5 Issues encounter 

Issue: original implementation caused pings that from Employees to File server to start to fail 

Cause: system no longer was dependent on ACL rules and was following ZFW which also had a default deny policy and originally only 1 Zone pair between HR and Server was made, so communication from Employees which was not in a zone was automatically dropped. 

Solution: Added new Zone : EMPLOYEE-ZONE, class map : EMPLOYEE-TO-SERVER, policy group : EMPLOYEE-POLICY and zone pair : EMPLOYEE-TO-SERVER-PAIR.

Lesson Learned: When employing ZFW it is important to make sure Every end point is attached to a zone, as well as its group of access rules, polies and its specific allowed pairs, otherwise connection are automatically dropped. In ZFW proper configuration is necessary for maintaining connectivity. 

## 6 Comparisons : ACL vs ZFW

<img width="698" height="150" alt="image" src="https://github.com/user-attachments/assets/10f82584-4a0f-4409-8429-25d4ef1b812e" />

For a production network after this project i would choose to use a ZFW. Although it may be more complex to configure a ZFW, it allows us to group up end points into zones which would allow for easier management and as it is not reliant on direction or explicit return rules it makes it easier to add more policy rules and implement them which is ideal for any system that should account for scaling upwards.

## 7 Difference in Production
in a real world environment i would add :
+ Use a dedicated firewall appliance as opposed to a routed based ZFW
+ Use SIEM for centralized logging of denied traffic alerts
+ Adopt a principle of least priviledge (remove permit any any )

## 8 Skills Demonstrated 
+ Network segmentation and subnet design
+ Standard and extended ACL configuration and troubleshooting
+ Cisco IOS CLI configuration (routing, interfaces, security features)
+ Zone-Based Policy Firewall design and implementation
+ Documentation of security policy from business requirement to technical control



