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



