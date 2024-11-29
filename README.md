# Task Management API Challenge

Three services in this system:

__Authenticator__: https://github.com/kiennt9696/authenticator.git

__Safekeeper__: https://github.com/kiennt9696/safekeeper.git

__Task API Management__: https://github.com/kiennt9696/task.git



## 1. Authentication & Authorization with Role-Based Access Control (RBAC)
### 1.1 Authentication & Authorization Flow
In this assignment, I use a basic authentication with username and password 
for ease of demonstration. Later we can integrate any OIDC provider with
this system for authorization with RBAC. The flow is as follows:

![](images/oauth2_flow.png)

__Authenticator__: is the authentication service, verifies users' username & password
then return a session token that represents a user login session.

__Safekeeper__: is the authorization server, receives a session token from __Client__,
verifies then grants _access token_ if RBAC is satisfied.

I use Oauth2 with Authorization Code grant. It helps to manage clients 
that want to be authorized via our service and make this flow more secured.
In my implementation, I prefer this flow but simplify it to focus more on RBAC,
which is the goal of this assignment.

### 1.2 Database design for RBAC

![](images/authentication_authorization_flow.png)
Tables and their meanings:

- __User__: store user info

- __Client__: store clients registered to our system, this is used for verification in oauth2 flow

- __Role__: manage roles in our system

- __Permission__: define permissions that __Role__ may have

- __PermissionObject__: define kind of resources that users want to access, i.e Task in this assignment

- __PermissionAction__: manage actions that can be operated on resources (__PermissionObject__), i.e read, edit, delete, create, etc.

- __Scope__: represent a specific operation on resources 

- __AuditTable__: an abstract table, not a real table in database, is used to track audit info on data created/updated time and who did it and so on. 

Relationship among them:

- a __User__ may be assigned to many __Role__
- a __Role__ may contains a group of __User__
- a __Permission__ may point to multiple resources (__PermissionObject__), there may be multiple __PermissionAction__ a resource,
therefore multiple __Scope__
  
- a __Scope__ may be applied to multiple __PermissionObject__, therefore multiple __Permission__

For example, with this design, a user may have multiple roles: Student, Librarian. Librarian has access
to Books and Borrower. For a book, one who owns it may lend the book or sell it. Let's say Alice has permission with
action "sell" of resource "book" then she can lend the book to others as well. Sometimes, an action on a resource
may imply some other operations. That's why I introduce a fine-grained level of access control with scopes. In the above example,
for action (sell) on resource (book) it has 2 scopes: sell:book, lend:book. With each scope, a separate logic business is
performed. This architecture is extendable and may be applied to complicated systems.

### 1.3 How this design work with Task Management System
We have 2 roles:
- Employee: 
  - view assigned tasks (basic_view → task)
  - update assigned task status (basic_update → task) 
- Employer:
  - view all tasks (super_view → task)
  - create tasks (super_create → task)
  - assign tasks (super_update → task)
    
The problem now is as modeled as follows. There are basic permissions on resource Task:

| Permission | object     | action     | scope      |
|------------|------------|------------|------------|
| view task basic    | task    | view    | read:task     |
| view task advanced    | task    | su_view    |su_read:task     |
| update status of assigned tasks only    | task    | update    |edit:task     |
| assign tasks for others    | task    | su_update    |su_edit:task     |

Let's fulfill this table for other business requirements. When the system grows and, features become complex
we can extend multiple scope for one action.

The service Saftekeeper follows these rules to determine where it will grant access token to client when they request it or not.
The authorization combining oauth2 and RBAC now is as follows.
![](images/oauth2_with_rbac.png)

Note: In my implementation, RBAC and Oauth2 flow are implemented in the same project __Safekeeper__. Later we can decompose this to scale out if needed.  

So far, the process I have mentioned is just a half of the final solution for this problem. 
The next part is how Task API service interacts with __scopes__ to accommodate the problem requirements.

In Task API service, now, it defines APIs that perform the business logic functions, such as view assigned tasks only, view all tasks, etc.
However, for each endpoint of these APIs, it is assigned with a specific scope for each function as I mentioned in the above permission table.
The Task API service then needs to validate the access token first to check whether it is a valid token then includes the right scope for the endpoint
before allowing function execution.

![](images/scope_validation.png)

##2. Task Management Flow
###2.1 Database design
![](images/task_management_db.png)

Tables and meaning:
- __Task__: the task table, stores info about task.
- __TaskType__: type of task such as bug, feature, task
- __TaskSeverity__: severity of task such as low, medium, high, critical
- __TaskStatus__: status of task such as todo, in progress, pending, completed
- __Workflow__: a workflow to manage statuses of task
- __Transition__: store the rules how statuses changes such as todo -> in progress 
  
Relationship among them:
- each __Task__ has a specific task type, task severity, task status at a time. 
- each task type is corresponding to a workflow. For example, regular tasks follows flow (todo -> in progress -> completed) 
but a task of "bug" uses another different flow such as (to do -> open - close -> reopen). This means workflow is changed when task type is changed.
- a workflow contains multiple transitions: A-> B, B-> C, C -> A.
- __TaskStatus__ is changed following its workflow.

The workflow I built for this system:

![](images/workflow.png)


There are 4 statuses: To Do, In Progress, Pending, Completed.
The common workflow is: 

To Do → In Progress → Completed. 

In some case,
some incidents happen or nonworking days, the task is changed to "Pending" but only
Employer has this permission, so I put a field called __approver__ in __Transition__ table to handle 
this situation.

###2.2 Functions and API design in the system
#### 2.2.1 Add user
```JSON
POST http://127.0.0.1:5000/v1/user
payload: {
    "username": "kiennt96",
    "password": "123456@A",
    "firstname": "Nguyen",
    "lastname": "Kien",
    "email": "kiennt9696@gmail.com",
    "phone": "0969716530"
}
```
#### 2.2.2 Login
```JSON
POST http://127.0.0.1:5000/v1/login
payload: {
    "username": "kiennt96",
    "password": "123456@A"
}
response: {
    "message": "Login successfully",
    "session_token": "eyJ0eXAiOiJKV1QiLCJhbGciOiJSUzI1NiJ9.eyJpc3MiOiJhdXRoZW50aWNhdG9yIiwidXNlcm5hbWUiOiJraWVubnQ5NiIsInN1YiI6IjI5MDI4NzAyLTg3ZmQtNGJjOS04YzM2LTMyMzI2MjI1ZWRhMiIsImV4cCI6MTczMjg5MzA3OH0.A446GKZG1Qy6jFnW1UnkaxfKlrypbtyhXLlPayHSzbR01UBYarRO9mCGaV12Y3BKqn8nNzUa8pOlqnEzuXh06RYmA7SC0ATL4AGWjZk6EPhdjd9iCCpgawaTQRsJb2AKqrLBWNRtmIgH5R7Zx857z9mT-ztBsu25iTPfapOjBDA"
}
```
#### 2.2.3 Get access token for scopes
![img.png](images/img.png)
```JSON
POST http://127.0.0.1:5001/v1/authorize
headers: {
    "Authorization": "session_token"
}
payload: {
    "scopes": "read:task,edit:task"
}
```
### 2.2.4 Get assigned task only (Employee)
![img_8.png](images/img_8.png)
```JSON
POST http://127.0.0.1:5002/v1/task/own
headers: {
    "Authorization": "Bearer access_token"
}
payload {
    "start_date": "2020-10-10 00:00:00",
    "end_date": "2024-12-12 00:00:00",
    "counting": true,
    "_from": 0,
    "_size": 100,
    "sort": "-created"
}
response: {
    "count": 2,
    "tasks": [
        {
            "assignee": "kiennt96",
            "created_at": "2024-11-29T07:48:57.533819",
            "creator": null,
            "description": null,
            "due_date": null,
            "id": "063c7cdb-5916-4a2c-afb3-234c2c852f7f",
            "title": "test",
            "updated_at": "2024-11-29T07:48:57.533819",
            "updated_by": null
        },
        {
            "assignee": "kiennt96",
            "created_at": "2024-11-29T07:48:57.528106",
            "creator": "kiennt96",
            "description": "Migrate data",
            "due_date": null,
            "id": "1",
            "title": "Migrate database to new place",
            "updated_at": "2024-11-29T07:48:57.528106",
            "updated_by": null
        }
    ]
}
```
Error when using access token with wrong scopes:

![img_2.png](images/img_2.png)
#### 2.2.5 Get all tasks (Employer)
![img_9.png](images/img_9.png)
```JSON
POST http://127.0.0.1:5002/v1/task/all
headers: {
    "Authorization": "Bearer access_token"
}
payload {
    "start_date": "2020-10-10 00:00:00",
    "end_date": "2024-12-12 00:00:00",
    "counting": true,
    "_from": 0,
    "_size": 100,
    "query": {
        "assignee": "kiennt96",
        "status_id": 4 # completed tasks
    },
    "sort": "-created_at,status_id"
}
```
#### 2.2.6 Update status of assigned tasks only (Employee)
![img_10.png](images/img_10.png)
```JSON
POST http://127.0.0.1:5002/v1/tasks/063c7cdb-5916-4a2c-afb3-234c2c852f7f
headers: {
    "Authorization": "Bearer access_token"
}
payload {
    "status_id": 2
}
```
#### 2.2.7 Assign tasks for others (Employer)
![img_11.png](images/img_11.png)
```JSON
POST http://127.0.0.1:5002/v1/task/assignment
headers: {
    "Authorization": "Bearer access_token"
}
payload {
    "assignee": "kiennt96"
}
```
#### 2.2.8 Create tasks (Employer)
![img_12.png](images/img_12.png)
```JSON
POST http://127.0.0.1:5002/v1/task
headers: {
    "Authorization": "Bearer access_token"
}
Payload: {
    "title": "Upgrade database postgres to v17",
    "type_id": 1,
    "assignee": "kiennt67",
    "description": "Upgrade database postgres to v17",
    "due_date": "2025-01-01 00:00:00"
}
```
#### 2.2.9 View employee task summary report (Employer)
![img_13.png](images/img_13.png)
```JSON
POST http://127.0.0.1:5002/v1/task/summary
headers: {
    "Authorization": "Bearer access_token"
}
payload {
    "start_date": "2020-10-10 00:00:00",
    "end_date": "2024-10-10 00:00:00",
    "_from": 0,
    "size": 100
}
response {
    "report": [
        ["kiennt67",2,0],
        ["kiennt96",2,0]
    ]
}
```

###3. Core processing solved technical problems

#### 3.1 Check user has some permissions via scopes (RBAC logic core)
Query to verify when Client ask for an access token for some scopes
```sql
SELECT COUNT(DISTINCT scope.id)
FROM scope
JOIN permission_scope ON scope.id = permission_scope.scope_id
JOIN permission ON permission.id = permission_scope.permission_id
JOIN role_permission ON role_permission.permission_id = permission.id
JOIN user_role ON user_role.role_id = role_permission.role_id
WHERE user_role.user_id = :user_id
  AND scope.name IN (:scopes)
```
Index to speedup query:
```sql
CREATE INDEX idx_user_role_user_id ON user_role(user_id);
CREATE INDEX idx_role_permission_role_id ON role_permission(role_id);
CREATE INDEX idx_permission_scope_permission_id ON permission_scope(permission_id);
CREATE INDEX idx_scope_name ON scope(name);
```
#### 3.2 Query tasks and Task report summary
```sql
SELECT 
    user.username, 
    COUNT(task.id) AS total_tasks, 
    COUNT(CASE WHEN task.status_id = :COMPLETED THEN 1 END) AS completed_tasks
FROM user
LEFT OUTER JOIN task ON user.username = task.assignee
WHERE user.role = :EMPLOYEE_ROLE
GROUP BY user.username
ORDER BY (Employer's fields)
OFFSET :_from ROWS
FETCH NEXT :size ROWS ONLY;
```
Indices to speed up the query
```sql
CREATE INDEX idx_task_assignee ON Task(assignee);
CREATE INDEX idx_task_status_id ON Task(status_id);
CREATE INDEX idx_user_username ON User(username);
CREATE INDEX idx_user_role ON User(role); # This table is different from table User in Authenticator  

CREATE INDEX idx_task_compound ON Task(assignee, status_id);
```

In my design Database for Authentication, Authorization and Task API are dependent for ease of scaling when system grows up.
Therefore, table User and table Task lies on different databases, some validation can be done in application layer.
I do not directly join Task with User and other RBAC tables to filter if they are Employee or not. Instead, 
I use a brief precomputing table User in Task Service's Database with enough info (username, role) to join then compute
summary report efficiently. I use CDC mechanism to reflect changes from real User database to User in Task Service.
However, when data grows up by time this query increases load and decrease system performance even with indexing.
I suggest the following approaches when facing that:
- Using database materialized views 
- Precompute results, accept eventual consistency
- Using cache
- Pagination
- Partition data

Depending on problems let's combine these to solve it.

#### 3.3 Why using scope based RBAC, not querying RBAC Database for every request? How to deal with staling permission with my approach?
Why using scope based RBAC, not querying RBAC Database for every request?

The answer is querying RBAC DB for every single request is expensive. I suggest re-verifing it only with critical functions.

How to deal with staling permission with my approach?

With JWT access token, it's stateless and in use until expired. Hence, in my design login session token (login token) has long expiration time,
but access token's is short, i.e 5 - 10 minutes. Along with that Client should remove or update user access token when 
there are users' role changes. Our system should be event-driven with Kafka queue.

#### 3.4 System performance with big data over time
When data grows up, especially with SQL database, Postgres as I use in this, system performance may go down significantly.
The solutions I suggest includes:
- Shard data into multiple nodes (this may introduce some overheads on application layer)
- Adding replica for read sharing load
- Caching
- Archival Service: for this system we prioritize tasks with task not completed. For completed ones, 
  they are rarely modified and eventual consistency is acceptable so let's move them to
another database such as Cassandra. This makes Postgres performance improved and stable. 


