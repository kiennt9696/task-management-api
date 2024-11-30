# Task Management API Challenge

Three services in this system:

__Task API Management__: https://github.com/kiennt9696/task.git

__Safekeeper__: https://github.com/kiennt9696/safekeeper.git

__Authenticator__: https://github.com/kiennt9696/authenticator.git

Common package is used for 3 projects: https://github.com/kiennt9696/common-utils.git

# Table of Contents
- [1. Authentication & Authorization with Role-Based Access Control (RBAC)](#1-authentication---authorization-with-role-based-access-control--rbac-)
  * [1.1 Authentication & Authorization Flow](#11-authentication---authorization-flow)
  * [1.2 Database design for RBAC](#12-database-design-for-rbac)
  * [1.3 How this design work with Task Management System](#13-how-this-design-work-with-task-management-system)
- [2. Task Management Flow](#2-task-management-flow)
  * [2.1 Database design](#21-database-design)
  * [2.2 Functions and API design in the system](#22-functions-and-api-design-in-the-system)
    + [2.2.1 Add user](#221-add-user)
    + [2.2.2 Login](#222-login)
    + [2.2.3 Get access token for scopes](#223-get-access-token-for-scopes)
  * [2.2.4 Get assigned task only (Employee)](#224-get-assigned-task-only--employee-)
    + [2.2.5 Get all tasks (Employer)](#225-get-all-tasks--employer-)
    + [2.2.6 Update status of assigned tasks only (Employee)](#226-update-status-of-assigned-tasks-only--employee-)
    + [2.2.7 Assign tasks for others (Employer)](#227-assign-tasks-for-others--employer-)
    + [2.2.8 Create tasks (Employer)](#228-create-tasks--employer-)
    + [2.2.9 View employee task summary report (Employer)](#229-view-employee-task-summary-report--employer-)
- [3. Core processing and solved technical problems](#3-core-processing-and-solved-technical-problems)
  * [3.1 Check user has some permissions via scopes (RBAC logic core)](#31-check-user-has-some-permissions-via-scopes--rbac-logic-core-)
  * [3.2 Query tasks and Task report summary](#32-query-tasks-and-task-report-summary)
  * [3.3 Why using scope based RBAC, not querying RBAC Database for every request? How to deal with staling permission with my approach?](#33-why-using-scope-based-rbac--not-querying-rbac-database-for-every-request--how-to-deal-with-staling-permission-with-my-approach-)
  * [3.4 System performance with big data over time](#34-system-performance-with-big-data-over-time)
- [4. Technologies and Patterns](#4-technologies-and-patterns)
  * [4.1 Language and frameworks](#41-language-and-frameworks)
  * [4.2 Database](#42-database)
  * [4.3 Project structures and layers](#43-project-structures-and-layers)
  * [4.4 Unittest and coverage](#44-unittest-and-coverage)
  * [4.5 Dockerization and deployment](#45-dockerization-and-deployment)
  * [4.6 Others](#46-others)
  

## Overall System Architecture

![](images/block_architecture.png)

## 1. Authentication & Authorization with Role-Based Access Control (RBAC)
### 1.1 Authentication & Authorization Flow
In this assignment, I use a basic authentication with username and password 
for ease of demonstration. Later on, we can integrate any OIDC provider with
this system for authorization with RBAC. The flow is as follows:

![](images/oauth2_flow.png)

__Authenticator__: is the authentication service, verifies users' username & password
then return a session token that represents a user login session.

__Safekeeper__: is the authorization server, receives a session token from __Client__,
verifies then grants _access token_ if RBAC is satisfied.

I use __Oauth2__ with __Authorization Code__ grant. It helps to manage clients 
that want to be authorized via our service and make this flow more secured.
In my implementation, I prefer this flow but simplify it to focus more on RBAC,
which is the primary goal of this assignment.

### 1.2 Database design for RBAC

![](images/authentication_authorization_flow.png)
Tables and their meanings:

- __User__: stores user info

- __Client__: stores clients registered to our system, this is used for verification in oauth2 flow

- __Role__: manages roles in our system

- __Permission__: defines permissions that __Role__ may have

- __PermissionObject__: defines kind of resources that users want to access, i.e Task in this assignment

- __PermissionAction__: manages actions that can be operated on resources (__PermissionObject__), i.e read, edit, delete, create, etc.

- __Scope__: represents a specific operation on resources 

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
There are 2 roles:
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

A role now may have multiple permissions

| Role | view task basic     |  view task advanced    |  update status of assigned tasks only     | assign tasks for others|
|------------|------------|------------|------------|-----------|
| Employee  |    x   |     | x     |        |
| Employer    | x    | x    | x     |     x |

Users then are assigned to roles

| User | Employee     |  Employer    |
|------------|------------|------------|
| Alice  |       |  x   | 
| Bob    |  x  |     |

Let's fulfill these tables for other business requirements. When the system grows up, features become complex
we can extend multiple scopes for one action.

Let's see how data is stored in database

![img_4.png](img_4.png)
![img_5.png](img_5.png)
![img_6.png](img_6.png)
![img_7.png](img_7.png)
![img_8.png](img_8.png)
![img_9.png](img_9.png)

Service __Saftekeeper__ follows these rules to determine where it will grant access token to client when they request it or not.
The authorization combining __oauth2__ and __RBAC__ now is as follows.

![](images/oauth2_with_rbac.png)

Note: In my implementation, RBAC and Oauth2 flow are implemented in the same project __Safekeeper__. Later, we can decompose this to scale out if needed.  

So far, the process I have mentioned is just a half of the final solution for this problem. 
The next part is how Task API service interacts with __scopes__ to accommodate the problem requirements.

In __Task API service__, now, it defines APIs that perform the business logic functions, such as view assigned tasks only, view all tasks, etc.
However, __for each endpoint of these APIs, it is assigned with a specific scope for each function__ as I mentioned in the above permission table.
__The Task API service then needs to validate the access token first to check whether it is a valid token then includes the right scope for the endpoint
before allowing function execution.__

![](images/scope_validation.png)

## 2. Task Management Flow
### 2.1 Database design

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
- each task type is corresponding to a workflow. For example, regular tasks follows flow (todo → in progress → completed) 
but a task of "bug" uses another different flow such as (to do → open → close → reopen). This means workflow is changed when task type is changed.
- a workflow contains multiple transitions: A → B, B → C, C → A.
- __TaskStatus__ is changed following its workflow.

The workflow I built for this system:

![](images/workflow.png)


There are 4 statuses: __To Do__, __In Progress__, __Pending__, __Completed__.
The common workflow is: 

To Do → In Progress → Completed. 

In case, some incidents happen or nonworking days, the task is changed to "Pending" but only
Employer has this permission, so I put a field called __approver__ in __Transition__ table to handle 
this situation.

![img_3.png](img_3.png)

### 2.2 Functions and API design in the system
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

## 3. Core processing and solved technical problems

### 3.1 Check user has some permissions via scopes (RBAC logic core)
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
### 3.2 Query tasks and Task report summary
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

In my design, databases for Authentication, Authorization and Task API are dependent for ease of scaling when system grows up.
Therefore, table User and table Task lies on different databases, some validation can be done in application layer.
I do not directly join Task with User and other RBAC tables to filter if they are Employee or not. Instead, 
I use a brief precomputing table User in Task Service's database with enough info (username, role) to join then compute
summary report efficiently. I use CDC mechanism to reflect changes from real User database to User in Task Service.
However, when data grows up by time this query increases load and decrease system performance even with indexing.
I suggest the following approaches when facing that:
- Using database materialized views 
- Precompute results, accept eventual consistency
- Using cache
- Pagination
- Partition data

Depending on problems let's combine these to solve it.

### 3.3 Why using scope based RBAC, not querying RBAC Database for every request? How to deal with staling permission with my approach?
_Why using scope based RBAC, not querying RBAC Database for every request?_

The answer is querying RBAC DB for every single request is expensive. I suggest re-verifing it only with critical functions.

_How to deal with staling permission with my approach?_

With JWT access token, it's stateless and in use until expired. Hence, in my design login session token (login token) has long expiration time,
but access token's is short, i.e 5 - 10 minutes. Along with that Client should remove or update user access token when 
there are users' role changes. Our system should be event-driven with Kafka queue so that it can notify others on data changes. Along with that with Oauth2 flow, Client must be verified is a security plus.   

### 3.4 System performance with big data over time
When data grows up, especially with SQL database, Postgres as I use in this, system performance may go down significantly.
The solutions I suggest includes:
- Sharding data into multiple nodes (this may introduce some overheads on application layer)
- Adding replica for read sharing load
- Caching
- __Archival Service__: for this system we prioritize tasks with task not completed. For completed ones, 
  they are rarely modified and eventual consistency is acceptable so let's move them to
another database such as __Cassandra__. This makes Postgres performance improved and stable. 
  
## 4. Technologies and Patterns
### 4.1 Language and frameworks
I use __Python Flask__ with __Connexion__ to leverage defining API endpoints, input validation, and documentation in a __OpenAPI__ specification file.
### 4.2 Database
I use __Postgres__ for storage, combining with __SQLAchemy ORM__.
### 4.3 Project structures and layers
```html
│   app.py
│   extension.py
│   __init__.py
│   __main__.py
│
├───controllers
│       healthz.py
│       task.py
│       __init__.py
│
├───infras
│   │   __init__.py
│   │
│   └───db
│           connection.py
│           __init__.py
│
├───interfaces
│   │   __init__.py
│   │
│   └───repositories
│           task.py
│           workflow.py
│           __init__.py
│
├───middlewares
│       token_validator.py
│       __init__.py
│
├───models
│       __init__.py
│
├───openapi
│       swagger.yaml
│
├───repositories
│       task.py
│       workflow.py
│       __init__.py
│
├───schemas
│       schema.py
│       __init__.py
│
├───scripts
│       crawl_data.py
│       index.sql
│       init_db_tables.sql
│       init_model.py
│       init_workflow.py
│       workflow_base_data.json
│
├───services
│       task.py
│       workflow.py
│       __init__.py
│
└───tests
    │   __init__.py
    │
    ├───controllers
    │       test_task_controller.py
    │
    ├───helpers
    │       test_util.py
    │
    ├───repositories
    │       test_task_repository.py
    │       test_workflow_repository.py
    │
    └───services
            test_task_service.py
            test_workflow_service.py

```
I split my project into 5 main layers:
1. __Routing and validation__ request params are handled by __Connexion__ with __OpenAPI__. 
   The security checking for matching scope is also handled here where I injected a token validator handler in __middlewares__.
   
2. __Controller Layer__: This layer receives requests, extracts params and calls __Service Layer__ to perform the right business functions.
3. __Service Layer (Use case)__: This layer handles all business logic functions such as verifying workflow, changing task statuses via interacting with __Repository__ to perform read/write to databases or external systems.
   In this layer, I use __Marshmallow__ to dump data to json before returning data to clients
4. __Repository Layer__: This layer only execute naive CRUD operations to database without any complicated logic.
5. __Infras__: Handle connections to a specific kind of database, i.e Postgres in my case.

Each layer does not interact directly with each other but interfaces that are defined in package __interfaces__. In this
way it helps the code well-organized and easy to mock when writing tests using DI.  

Other components including:
- __extensions__: setup 2 middlewares (logger & __prometheus__ for collecting performance metrics in service).
- __models and schemas__: define models and schemas for tables in database for data serialization and deserialization.
- __scripts__: script files, data for initializing base data for system such as task workflow, RBAC. This is useful materials when packing the service using Ansible or similar tools.
- __tests__: unittests/tests for every thing.
- __alembic__: for migrating database changes.

I also set up a common package that contains common functions on token, api_error_handler used across all of my 3 projects.

![img.png](img.png)

Let's say, I have an api error handler in _common-utils_:

![img_1.png](img_1.png)

then, it is plugged in any Flask project to use:

![img_2.png](img_2.png)

This way I can organize my code efficiently.

### 4.4 Unittest and coverage
Run unittest command:
```html
python -m pytest -sv --cov-report xml:test_coverage/coverage.xml  --cov-report term-missing --cov=task task/tests
```
My code coverage:
```html
Name                                                  Stmts   Miss  Cover   Missing
-----------------------------------------------------------------------------------
task\__init__.py                                         50     28    44%   25, 35-50, 60-62, 71-84, 92-94, 98-100
task\__main__.py                                          7      7     0%   1-13
task\app.py                                               3      3     0%   1-5
task\controllers\__init__.py                              0      0   100%
task\controllers\healthz.py                               4      4     0%   1-6
task\controllers\task.py                                 38      0   100%
task\extension.py                                         7      0   100%
task\infras\__init__.py                                   0      0   100%
task\infras\db\__init__.py                                0      0   100%
task\infras\db\connection.py                             21      9    57%   13-14, 17-18, 21, 24-26, 29
task\interfaces\__init__.py                               0      0   100%
task\interfaces\repositories\__init__.py                  0      0   100%
task\interfaces\repositories\task.py                     16      4    75%   13, 17, 21, 27
task\interfaces\repositories\workflow.py                  6      1    83%   12
task\middlewares\__init__.py                              0      0   100%
task\middlewares\token_validator.py                      11     11     0%   1-14
task\models\__init__.py                                  70      3    96%   108-110
task\repositories\__init__.py                             5      0   100%
task\repositories\task.py                                35      2    94%   27, 36
task\repositories\workflow.py                            12      0   100%
task\schemas\__init__.py                                  0      0   100%
task\schemas\schema.py                                    6      0   100%
task\services\__init__.py                                 5      0   100%
task\services\task.py                                    56      1    98%   98
task\services\workflow.py                                13      0   100%
task\tests\__init__.py                                   26      0   100%
task\tests\controllers\test_task_controller.py           44      0   100%
task\tests\helpers\test_util.py                          37      0   100%
task\tests\repositories\test_task_repository.py          49      0   100%
task\tests\repositories\test_workflow_repository.py      24      0   100%
task\tests\services\test_task_service.py                 73      0   100%
task\tests\services\test_workflow_service.py             29      0   100%
-----------------------------------------------------------------------------------
TOTAL                                                   647     73    89%
Coverage XML written to file test_coverage/coverage.xml

=========================================================================== 28 passed, 10 warnings in 1.46s ===========================================================================

```
### 4.5 Dockerization and deployment
Dockerfile
```dockerfile
FROM python:3.8.0b1-slim-stretch

RUN mkdir -p /usr/src/app
WORKDIR /usr/src/app

COPY requirements.txt /usr/src/app/

RUN pip3 install --no-cache-dir -r requirements.txt --upgrade

COPY . /usr/src/app
ENV prometheus_multiproc_dir="/tmp"
RUN chmod +x run.sh

CMD ["./run.sh"]
```
In production environment, lets's deploy app with __Gunicorn__ and __WSGI__.
```shell
run.sh
-----------
#!/bin/bash

worker=${WORKER:-1}
port=${PORT:-8082}
timeout=${TIMEOUT:-60}

gunicorn -c prometheus_config.py --bind 0.0.0.0:${port} -k gevent -w $worker --timeout $timeout wsgi

```
Build image (set PROXY if needed or just remove it):
```shell
docker image build --network=host --build-arg http_proxy=$(PROXY) --build-arg https_proxy=$(PROXY) --build-arg NO_PROXY=127.0.0.1 -t task-management-api:1.0.0 .
```
Docker-compose setup
```shell
version: '2'

services:
    task_api:
      container_name: task-api
      image: task-management-api:1.0.0
      volumes:
        - ./config.yaml:/usr/src/app/config.yaml
        - ./logger.conf:/usr/src/app/logger.conf
        - ./logs:/usr/src/app/logs
      environment:
        PORT: 8080
        WORKER: 1
        METRICS_PORT: 18080
        prometheus_multiproc_dir: /tmp

      ports:
        - 8086:8080
```
Deploy using docker-compose
```shell
docker-compose up -d
```
### 4.6 Others
- __Pagination__ is applied to every API.
- I always implement a __healthz api__ for a service to monitor service's health.
- __Prometheus__ is used to collect metrics on service performance. 
