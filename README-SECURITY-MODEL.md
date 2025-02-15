   Licensed to the Apache Software Foundation (ASF) under one
   or more contributor license agreements.  See the NOTICE file
   distributed with this work for additional information
   regarding copyright ownership.  The ASF licenses this file
   to you under the Apache License, Version 2.0 (the
   "License"); you may not use this file except in compliance
   with the License.  You may obtain a copy of the License at

     http://www.apache.org/licenses/LICENSE-2.0

   Unless required by applicable law or agreed to in writing,
   software distributed under the License is distributed on an
   "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
   KIND, either express or implied.  See the License for the
   specific language governing permissions and limitations
   under the License.

# README for Apache Fortress Security Model
___________________________________________________________________________________
## Table of Contents

 * Document Overview
 * Understand the security model of Apache Fortress Rest
 * 1. Java EE security
 * 2. Apache CXF's **SimpleAuthorizingInterceptor**
 * 3. Apache Fortress **ARBAC02 Checks**
 * The list of APIs that enforce ARBAC role range and OU checks.
___________________________________________________________________________________

## Document Overview

 Provides a description of the various security mechanisms that are performed during Apache Fortress REST runtime operations.
___________________________________________________________________________________

## Understand the security model of Apache Fortress Rest

 * Apache Fortress Rest is a JAX-RS Web application that allows the Apache Fortress Core APIs to be called over an HTTP interface.
 * It deploys inside of any compliant Java Servlet container although here we'll be using Apache Tomcat.

### Apache Fortress Rest security model includes:

### TLS

 Nothing special or unique going on here.  Refer to the documentation of your servlet container for how to enable.

___________________________________________________________________________________
## 1. Java EE security

 * Apache Fortress Rest uses the [Apache Fortress Realm](https://github.com/apache/directory-fortress-realm) to provide Java EE authentication, coarse-grained authorization mapping the users and roles back to a given LDAP server.
 * The policy for Apache Fortress Rest is simple.  Any user with the **fortress-rest-user** role and correct credentials is allowed in.
 * The Fortress Rest interface uses HTTP Basic Auth tokens to send the userid/password.
___________________________________________________________________________________
## 2. Apache CXF's **SimpleAuthorizingInterceptor**

This policy enforcement mechanism maps RBAC roles to a given set of services.  The following table shows what roles map to which (sets of) services:

| service type      | fortress-rest-super-user | fortress-rest-admin-user | fortress-rest-review-user | fortress-rest-access-user | fortress-rest-deladmin-user | fortress-rest-delreview-user | fortress-rest-delaccess-user | fortress-rest-pwmgr-user | fortress-rest-audit-user | fortress-rest-config-user |
| ----------------- | ------------------------ | ------------------------ | ------------------------- | ------------------------- | --------------------------- | ---------------------------- | ---------------------------- | ------------------------ | ------------------------ | ------------------------- |
| Admin  Manager    | true                     | true                     | false                     | false                     | false                       | false                        | false                        | false                    | false                    | false                     |
| Review Manager    | true                     | false                    | true                      | false                     | false                       | false                        | false                        | false                    | false                    | false                     |
| Access Manager    | true                     | false                    | false                     | true                      | false                       | false                        | false                        | false                    | false                    | false                     |
| Delegated Admin   | true                     | false                    | false                     | false                     | true                        | false                        | false                        | false                    | false                    | false                     |
| Delegated Review  | true                     | false                    | false                     | false                     | false                       | true                         | false                        | false                    | false                    | false                     |
| Delegated Access  | true                     | false                    | false                     | false                     | false                       | false                        | true                         | false                    | false                    | false                     |
| Password  Manager | true                     | false                    | false                     | false                     | false                       | false                        | false                        | true                     | false                    | false                     |
| Audit  Manager    | true                     | false                    | false                     | false                     | false                       | false                        | false                        | false                    | true                     | false                     |
| Config  Manager   | true                     | false                    | false                     | false                     | false                       | false                        | false                        | false                    | false                    | true                      |

___________________________________________________________________________________
## 3. Apache Fortress **ARBAC02 Checks**

 Disabled by default.  To enable, add this to fortress.properties file and restart instance:

 ```concept
# Boolean value. Disabled by default. If this is set to true, the runtime will enforce administrative permissions and ARBAC02 DA checks:
is.arbac02=true

 ```

The ARBAC checks when enabled, include the following:

1. All service invocations perform an ADMIN permission check automatically corresponding with the exact service/API being called. 
 For example, the permission with an objectName: **org.apache.directory.fortress.core.impl.AdminMgrImpl** and operation name: **addUser** is automatically checked
 during the call to the **userAdd** service.   
 This means at least one ADMIN role must be activated for the user calling the service that has been granted the required permission.
 The entire list of permissions can be found here: [FortressRestServerPolicy](./src/main/resources/FortressRestServerPolicy.xml) along with a sample policy that can be used for testing.

2. Some services (listed below) perform an ARBAC role range check on the target RBAC role. 
 The Apache Fortress REST **roleAsgn**, **roleDeasgn**, **roleGrant** and **roleRevoke** services map to the **assignUser**, **deassignUser**, **grantPermission**, **revokePermission** Apache Fortress Core AdminMgr APIs respectively.
 During service dispatch of these APIs, the runtime will enforce ADMIN authority over the particular RBAC role that is being targeted in the HTTP request. 
 These checks are based on a (hierarchical) range of roles, for which the target role must fall inside.   
 For example, the following top-down contains a sample RBAC role hierarchy for a fictional software development organization:

 ```
        CTO
         |
     |         |
    ENG        QC
   |   |     |    |   
  E1   E2   Q1    Q2
     |         |
    DA        QA
         |
         A
 ```
    
 Here a role called *CTO* is the highest ascendant in the graph, and *A* is the lowest descendant. In a top-down role hierarchy, privilege increases as we descend downward.  So a person with role *A* inherits all that are above.

 In describing a range of roles, *beginRange* is the lowest descendant in the chain, and *endRange* the highest. Furthermore a bracket, '[', ']', indicates inclusiveness with an endpoint, whereas parenthesis, '(', ')' will exclude a corresponding endpoint.

 Some example ranges that can be derived from the sample role graph above:

 * [A, CTO] is the full set: {CTO, ENG, QC, E1, E2, Q1, Q2, DA, QA, A}. 
 * (A, CTO) is the full set, minus the endpoints: {ENG, QC, E1, E2, Q1, Q2, DA, QA}. 
 * [A, ENG] includes: {A, DA, E1, E2, ENG}, 
 * [A, ENG) includes: {A, DA, E1, E2}. 
 * (QA, QC] has {Q1, Q2, QC} in its range.
 * etc... 

 For an administrator to be authorized to target an RBAC role in one of the specified APIs listed above, at least one of their activated ADMIN roles must pass the ARBAC role range test.  There are currently two roles 
 created by the security policy in this project, [FortressRestServerPolicy](./src/main/resources/FortressRestServerPolicy.xml), that are excluded from this type of check: 
 **fortress-rest-admin** and **fortress-core-super-admin**. 

 Which means they won't have to pass the role range test.  All others use the range field to define authority over a particular set of roles, in a hierarchical structure. 
                                         
3. Some APIs (listed below) do organization checks, matching the org on the ADMIN role with that on the target user or permission.  
 There are two types of organziations, User and Permission.  For example, de/assignUser(User, Role) will verify that the caller has an ADMIN role with a user org unit that matches the ou of the target user.  
 There is a similar check on grant/revokePermission(Role, Permission), verifying the caller has an activated ADMIN role with a perm org unit that matches the ou on the target permission.

### The list of APIs that enforce ARBAC role range and OU checks.

| API                            | Validate UserOU  | Validate PermOU | Range Check On Role | 
| ------------------------------ | ---------------- | ----------------| ------------------- | 
| AdminMgr.addUser               | true             | false           | false               | 
| AdminMgr.updateUser            | true             | false           | false               | 
| AdminMgr.deleteUser            | true             | false           | false               | 
| AdminMgr.disableUser           | true             | false           | false               | 
| AdminMgr.changePassword        | true             | false           | false               | 
| AdminMgr.resetPassword         | true             | false           | false               | 
| AdminMgr.lockUserAccount       | true             | false           | false               | 
| AdminMgr.unlockUserAccount     | true             | false           | false               | 
| AdminMgr.deletePasswordPolicy  | true             | false           | false               | 
| AdminMgr.assignUser            | true             | false           | true                | 
| AdminMgr.deassignUser          | true             | false           | true                | 
| AdminMgr.grantPermission       | false            | true            | true                | 
| AdminMgr.revokePermission      | false            | true            | true                | 


#### END OF README