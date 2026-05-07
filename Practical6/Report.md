# Practical 6: Securing Redis and MongoDB

**Authentication, Encryption, RBAC, and Security Audit**

**Course:** DBS302 — NoSQL Database Management


**Name:** Dupchu Wangmo

**Student ID:** 02230282

**Date:** 28 Apr 2026

## 1. Aim

To configure and verify authentication, encryption, and role-based access control for Redis and MongoDB, and to perform a basic security audit of the configured databases.

## 2. Objectives

By the end of this practical, the student should be able to:

- Enable password-based authentication and Access Control Lists (ACL) in Redis.
- Enable TLS encryption for Redis connections at the configuration and client level.
- Enable authentication and RBAC in MongoDB using built-in and custom roles.
- Enable TLS for MongoDB so that traffic is encrypted in transit.
- Execute a security audit checklist for both databases and record findings.

## 3. Theory

Three concepts underpin database security in this practical:

### 3.1 Authentication

Authentication is the process of verifying the identity of a client connecting to the database. In both Redis and MongoDB, this is done through usernames and passwords. Without authentication, anyone who can reach the database port can issue commands, which is a serious security risk.

### 3.2 Encryption (TLS)

Transport Layer Security (TLS) protects data while it travels across the network. Without TLS, an attacker on the same network can intercept passwords and data in plaintext. TLS uses certificates to verify identity and encrypts the entire communication channel between the client and server.

### 3.3 Role-Based Access Control (RBAC)

RBAC enforces the principle of least privilege: each user receives only the permissions strictly necessary for their job. In Redis this is achieved through ACL rules that restrict which keys a user may access and which commands they may run. In MongoDB this is achieved through built-in or custom roles that define allowed actions on specific databases and collections.


# Part A — Securing Redis

## A1. Procedure

### Step 1: Verify Redis installation

Confirmed Redis was installed and the server was running using:

```bash
redis-server --version
redis-cli ping
```


### Step 2: Create isolated working directory and config

To avoid modifying the system-wide Homebrew configuration, an isolated working directory was created and a custom `redis.conf` was written with ACL users defined.

```bash
mkdir -p ~/dbs302-practical6/redis/data
mkdir -p ~/dbs302-practical6/redis/tls
cd ~/dbs302-practical6/redis
```

![alt text](screenshots/fig2.png)

The custom `redis.conf` contained the following ACL configuration:


![alt text](screenshots/fig3.png)



### Step 3: Stop default Redis and start the isolated instance

```bash
brew services stop redis
redis-server ~/dbs302-practical6/redis/redis.conf
ps aux | grep redis-server | grep -v grep
tail -20 ~/dbs302-practical6/redis/redis.log
```

![alt text](screenshots/fig4.png)

### Step 4: Verify anonymous access is denied

With the default user disabled, an unauthenticated client should not be able to issue any commands.

```bash
redis-cli ping
```

![alt text](screenshots/fig5.png)

### Step 5: Connect as admin and inspect ACL configuration

```bash
redis-cli -u redis://admin:adminStrongPwd@127.0.0.1:6379

ACL WHOAMI
ACL LIST
SET adminkey "hello from admin"
GET adminkey
```

![alt text](screenshots/fig7.png)

### Step 6: Test app_user — RBAC denial demonstration

This step is the most important evidence of RBAC: a limited user must succeed at allowed operations and fail at forbidden ones.

```bash
redis-cli -u redis://app_user:appStrongPwd@127.0.0.1:6379

ACL WHOAMI
SET session:user123 "valid session data"   
GET session:user123                        
SET otherkey "should fail"                  
GET adminkey                                
FLUSHALL                                    
CONFIG GET maxmemory                        
```

![alt text](screenshots/fig8.png)

### Step 7: Enable TLS for Redis

Self-signed certificates were generated for the lab so that the Redis server could be reconfigured to require TLS connections.

**Generate the CA and server certificates:**

```bash
cd ~/dbs302-practical6/redis/tls

openssl genrsa -out ca.key 4096
openssl req -x509 -new -nodes -key ca.key -sha256 -days 365 \
  -out ca.crt \
  -subj "/C=BT/ST=Chukha/L=Phuntsholing/O=DBS302/OU=Lab/CN=redis-lab-ca"

openssl genrsa -out redis.key 4096
openssl req -new -key redis.key -out redis.csr \
  -subj "/C=BT/ST=Chukha/L=Phuntsholing/O=DBS302/OU=Lab/CN=localhost"
openssl x509 -req -in redis.csr -CA ca.crt -CAkey ca.key -CAcreateserial \
  -out redis.crt -days 365 -sha256
```
![alt text](screenshots/fig9.png)

**TLS lines added to redis.conf:**

```conf
port 0
tls-port 6379
tls-ca-cert-file /Users/<username>/dbs302-practical6/redis/tls/ca.crt
tls-cert-file /Users/<username>/dbs302-practical6/redis/tls/redis.crt
tls-key-file /Users/<username>/dbs302-practical6/redis/tls/redis.key
tls-auth-clients no
```

**Restart Redis and connect with TLS:**

```bash

redis-cli -p 6379 shutdown nosave   
redis-server ~/dbs302-practical6/redis/redis.conf


redis-cli -u redis://admin:adminStrongPwd@127.0.0.1:6379 ping

redis-cli --tls --cacert ~/dbs302-practical6/redis/tls/ca.crt \
  -u rediss://admin:adminStrongPwd@127.0.0.1:6379
ping
ACL WHOAMI
```

# Part B

## Theory

Three concepts underpin database security in this practical:

### 1 Authentication

Authentication is the process of verifying the identity of a client connecting to the database. In MongoDB, this is done through usernames and passwords stored in the `admin` database (or another authentication database). Without authentication, anyone who can reach port 27017 can issue commands, which is a serious security risk.

### 2 Role-Based Access Control (RBAC)

RBAC enforces the principle of least privilege: each user receives only the permissions strictly necessary for their job. MongoDB ships with built-in roles such as `read`, `readWrite`, and `userAdminAnyDatabase`, and also allows administrators to create custom roles that grant specific actions on specific resources (databases or collections). When `--auth` is enabled, MongoDB enforces these roles for every operation.

### 3 Encryption (TLS)

Transport Layer Security (TLS) protects data while it travels across the network. Without TLS, an attacker on the same network can intercept passwords and data in plaintext. TLS uses certificates to verify identity and encrypts the entire communication channel between the client and server. MongoDB supports TLS through the `--tlsMode requireTLS` option together with a server certificate and a Certificate Authority file.



##  Procedure

Requirement 

![alt text](screenshots/MongoDB-fig1.png)

### Step 1: Create isolated working directory

To keep the practical files separate from any system-wide MongoDB installation, an isolated working directory was created.

```bash
mkdir -p ~/dbs302-practical6/mongo/data
mkdir -p ~/dbs302-practical6/mongo/logs
mkdir -p ~/dbs302-practical6/mongo/tls
```

This produced three subdirectories: `data` for the database files, `logs` for the mongod log, and `tls` for the certificates created later.



### Step 2: Start MongoDB without authentication (initial setup)

Authentication was temporarily left off so the first admin user could be created. After that, authorization is enabled.

```bash
mongod --dbpath ~/dbs302-practical6/mongo/data \
  --logpath ~/dbs302-practical6/mongo/logs/mongod.log \
  --bind_ip 127.0.0.1 \
  --port 27017 \
  --fork
```

![alt text](screenshots/MongoDB-fig2.png)

### Step 3: Create the root admin user

Connected with mongosh:

```bash
mongosh --host 127.0.0.1 --port 27017
```

![alt text](screenshots/MongoDB-fig3.png)

Inside mongosh:

```javascript
use admin

db.createUser({
  user: "rootAdmin",
  pwd: "rootStrongPwd",
  roles: [
    { role: "userAdminAnyDatabase", db: "admin" },
    { role: "dbAdminAnyDatabase", db: "admin" },
    { role: "readWriteAnyDatabase", db: "admin" }
  ]
})
```
![alt text](screenshots/MongoDB-fig4.png)

### Step 4: Restart MongoDB with authentication enabled

The running mongod was stopped and restarted with the `--auth` flag, which enables RBAC enforcement for all subsequent connections.

```bash
# Find the running PID and stop it
lsof -i :27017
kill <PID>



# Restart with --auth
mongod --dbpath ~/dbs302-practical6/mongo/data \
  --logpath ~/dbs302-practical6/mongo/logs/mongod.log \
  --bind_ip 127.0.0.1 \
  --port 27017 \
  --auth \
  --fork
```

![alt text](screenshots/MongoDB-fig5.png)

![alt text](screenshots/MongoDB-fig6.png)

### Step 5: Verify authentication is enforced

**Test 1 — connect without credentials (should fail):**

```bash
mongosh --host 127.0.0.1 --port 27017
```
![alt text](screenshots/MongoDB-fig7.png)

This denial confirms that anonymous access is blocked.

**Test 2 — connect with valid credentials (should succeed):**

```bash
mongosh --host 127.0.0.1 --port 27017 \
  -u rootAdmin -p rootStrongPwd \
  --authenticationDatabase admin
```

![alt text](screenshots/MongoDB-fig8.png)

Authentication succeeded and the response confirms `rootAdmin` is logged in with the expected roles.


### Step 6: Create application database, custom role, and limited user

Still inside mongosh as `rootAdmin`:

```javascript
use myapp

db.runCommand({
  createRole: "myAppRole",
  privileges: [
    {
      resource: { db: "myapp", collection: "customers" },
      actions: ["find", "insert", "update", "remove"]
    }
  ],
  roles: []
})
```

![alt text](screenshots/MongoDB-fig9.png)

This custom role can perform only four operations on a single collection — a strict implementation of least privilege.

```javascript
db.createUser({
  user: "appUser",
  pwd: "appStrongPwd",
  roles: [ { role: "myAppRole", db: "myapp" } ]
})
```
![alt text](screenshots/MongoDB-fig10.png)

### Step 7: Test RBAC with appUser — denial demonstration

Connected as the limited user:

```bash
mongosh --host 127.0.0.1 --port 27017 \
  -u appUser -p appStrongPwd \
  --authenticationDatabase myapp
```

**Allowed operations (should succeed):**

```javascript
myapp> use myapp
switched to db myapp

myapp> db.customers.insertOne({ name: "Student One", city: "Phuntsholing" })

myapp> db.customers.find()

myapp> db.customers.insertOne({ name: "Test Student", city: "Thimphu" })

```

![alt text](screenshots/MongoDB-fig11.png)

**Denied operations (should fail):**

```javascript
myapp> db.orders.insertOne({ item: "should fail" })


myapp> use admin
switched to db admin

admin> db.system.users.find()

```
![alt text](screenshots/MongoDB-fig12.png)

The RBAC system correctly denied access to:

- A different collection (`orders`) within the same database
- A completely different database (`admin`)

`appUser` is restricted to the four allowed operations on `myapp.customers` only.



## 6. Conclusion

This practical demonstrated how authentication, role-based access control, and transport encryption together protect a MongoDB database from unauthorized access and network eavesdropping. Enabling `--auth` blocked anonymous clients from issuing any commands. Combining authentication with a custom role enforced the principle of least privilege at the database and collection level: `appUser` could read and write only the `myapp.customers` collection and was denied all other operations. Finally, requiring TLS ensured that credentials and data could not be intercepted on the network, since plain connections were refused outright.

The exercise reinforced that database security is layered: any single control on its own is insufficient, but used together they create a defensible setup that aligns with common best practices for NoSQL deployments.


## 7. Issues Encountered and Resolutions

During the practical, the following issues were encountered and resolved:

1. **MongoDB `--shutdown` flag not recognised** — newer versions of MongoDB (8.0) have removed this option. Resolved by using `lsof -i :27017` to find the running PID and stopping it with `kill <PID>` before restarting with the `--auth` flag.

2. **MongoDB "Address already in use" error** — a previous mongod instance was still running on port 27017 after a failed start, causing the new mongod to exit immediately with exit code 48. Resolved by identifying the PID via `lsof -i :27017` and terminating it with `kill <PID>` before retrying.

