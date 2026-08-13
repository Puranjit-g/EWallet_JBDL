# EWallet Microservices Project

This repository contains a simple event-driven e-wallet system built with Spring Boot, Spring Security, Spring Data JPA, MySQL, and Kafka. The project is organized as a multi-module Maven build and demonstrates how user onboarding, wallet creation, transaction initiation, wallet balance updates, and notification delivery can be split across separate services.

## Overview

The application is composed of these modules:

- `User-Service`
- `Wallet-Service`
- `Transaction-Service`
- `Notification-service`
- `utils`

At a high level:

1. A user is created in `User-Service`.
2. `User-Service` publishes a `USER_CREATED_FROM_CONSOLE` Kafka event.
3. `Wallet-Service` listens to that event and creates a wallet with a default balance.
4. `Notification-service` listens to the same event and sends a welcome email.
5. A logged-in user initiates a transaction through `Transaction-Service`.
6. `Transaction-Service` publishes a `TXN_INITIATED_TOPIC` event.
7. `Wallet-Service` validates wallets, updates balances, and publishes a `TXN_UPDATED_TOPIC` event.
8. `Transaction-Service` currently receives the update event but does not yet persist the final transaction status.

## Tech Stack

- Java 17 source level in Maven configuration
- Spring Boot `3.3.1`
- Spring Web
- Spring Security with HTTP Basic authentication
- Spring Data JPA
- MySQL
- Apache Kafka
- Lombok
- Mailtrap SMTP for email testing

## Repository Structure

```text
EWallet_JBDL-main/
├── pom.xml
├── User-Service/
├── Wallet-Service/
├── Transaction-Service/
├── Notification-service/
└── utils/
```

## Module Responsibilities

### `User-Service`

Responsibilities:

- Register or update users
- Store user records in MySQL
- Encode passwords with BCrypt
- Expose user details for authentication
- Publish a user-created Kafka event
- Seed a service account for inter-service authentication

Important endpoints:

- `POST /user/addUpdate`
- `GET /user/userDetails?contact=<contact>`

Port:

- `8081`

Important behavior:

- New users are always assigned the `USER` authority in the current implementation.
- On startup, the service creates a service user:
  - username/contact: `txn-service`
  - password: `txn-service`
  - authority: `SERVICE`

### `Wallet-Service`

Responsibilities:

- Create a wallet when a user-created event is consumed
- Assign a default initial wallet balance
- Validate sender and receiver wallets for transactions
- Debit the sender and credit the receiver
- Publish transaction result events

Port:

- `7071`

Default wallet balance on user creation:

- `50.0`

### `Transaction-Service`

Responsibilities:

- Authenticate end users
- Accept transaction initiation requests
- Persist initiated transactions
- Publish transaction initiation events
- Consume transaction update events

Port:

- `5051`

Important endpoint:

- `POST /txn/initTxn`

Request parameters:

- `receiver`
- `purpose`
- `amount`

Authentication:

- Requires a user authenticated with `USER` authority
- Uses HTTP Basic authentication

Inter-service dependency:

- Calls `User-Service` at `http://localhost:8081/user/userDetails`
- Authenticates to `User-Service` using `txn-service / txn-service`

### `Notification-service`

Responsibilities:

- Listen for user-created events
- Send welcome email notifications

Port:

- `6061`

SMTP provider configured in code:

- Mailtrap sandbox SMTP

### `utils`

Responsibilities:

- Shared constants for Kafka topic names and JSON keys
- Shared enums such as user identifier type

## Shared Kafka Topics

Defined in `utils/src/main/java/org/gfg/Utilities/CommonConstants.java`:

- `USER_CREATED_FROM_CONSOLE`
- `WALLET_CREATED_FROM_CONSOLE`
- `TXN_INITIATED_TOPIC`
- `TXN_UPDATED_TOPIC`

## Data Model

### User

Stored by `User-Service`.

Important fields:

- `pk`
- `contact`
- `email`
- `authorities`
- `password`
- `name`
- `address`
- `dob`
- `userType`
- `identifier`
- `userIdentifierValue`
- audit timestamps

Notes:

- `contact` is unique
- `email` is unique
- Spring Security uses `contact` as the username

### Wallet

Stored by `Wallet-Service`.

Important fields:

- `id`
- `userId`
- `contact`
- `balance`
- audit timestamps

### Transaction

Stored by `Transaction-Service`.

Important fields:

- `pk`
- `txnId`
- `amount`
- `sender`
- `receiver`
- `purpose`
- `status`
- audit timestamps

Transaction statuses defined in code:

- `PENDING`
- `INITIATED`
- `SUCCESS`
- `FAILED`

## Security Model

Authorities configured in properties:

- `USER`
- `ADMIN`
- `SERVICE`

Current access rules:

- `POST /user/addUpdate` is open to everyone
- `GET /user/userDetails` requires `SERVICE` or `ADMIN`
- `POST /txn/initTxn` requires `USER`

Authentication style:

- HTTP Basic
- CSRF disabled

## Required Local Infrastructure

You need these services running locally:

- MySQL on `localhost:3306`
- Kafka on `localhost:9092`

Database used by all services:

- `JBDL_70_EWALLET`

Default MySQL credentials currently hardcoded in properties:

- username: `root`
- password: `rootroot`

## Configuration Summary

### User-Service

From `application.properties`:

- port: `8081`
- database: `jdbc:mysql://localhost:3306/JBDL_70_EWALLET?createDatabaseIfNotExist=true`
- Kafka producer enabled

### Wallet-Service

- port: `7071`
- same MySQL database
- Kafka consumer and producer enabled
- default new-wallet balance: `50.0`

### Transaction-Service

- port: `5051`
- same MySQL database
- Kafka consumer and producer enabled through explicit config classes

### Notification-service

- port: `6061`
- Kafka consumer enabled
- SMTP host: `sandbox.smtp.mailtrap.io`

## How the Main Flow Works

### 1. User Registration

Call:

```http
POST /user/addUpdate
Content-Type: application/json
```

Example request body:

```json
{
  "name": "Rahul",
  "contact": "9999999999",
  "email": "rahul@example.com",
  "address": "Bangalore",
  "dob": "1998-10-21",
  "userIdentifier": "PAN",
  "userIdentifierValue": "ABCDE1234F",
  "password": "secret123"
}
```

What happens:

- User is saved in MySQL
- Password is BCrypt-encoded
- A user-created event is published to Kafka

### 2. Wallet Auto-Creation

`Wallet-Service` consumes the user-created event and:

- extracts `user_id` and `contact`
- creates a wallet
- sets the starting balance to `50.0`
- publishes a wallet-created event

### 3. Welcome Email

`Notification-service` consumes the same user-created event and:

- reads `name` and `email`
- sends a welcome email through Mailtrap SMTP

### 4. Transaction Initiation

Call:

```http
POST /txn/initTxn?receiver=8888888888&purpose=rent&amount=20
Authorization: Basic <base64(user-contact:user-password)>
```

What happens:

- `Transaction-Service` authenticates the sender
- creates a `Txn` record with status `INITIATED`
- generates a UUID transaction id
- publishes a transaction-initiated Kafka event

### 5. Wallet Balance Update

`Wallet-Service` consumes the transaction event and checks:

- sender wallet exists
- receiver wallet exists
- sender has enough balance

Outcomes:

- on success: sender balance is reduced and receiver balance is increased
- on failure: a failure message is created

Then it publishes a transaction update event with:

- `txnId`
- `status`
- `message`

## Running the Project

### Prerequisites

- JDK 17 or compatible Java installation
- Maven wrapper support
- MySQL running locally
- Kafka running locally

### Build all modules

From the project root:

```bash
./mvnw clean install
```

### Run services

Start each service from its module directory or from your IDE:

```bash
./mvnw spring-boot:run -pl User-Service
./mvnw spring-boot:run -pl Wallet-Service
./mvnw spring-boot:run -pl Transaction-Service
./mvnw spring-boot:run -pl Notification-service
```

Suggested startup order:

1. MySQL
2. Kafka
3. `User-Service`
4. `Wallet-Service`
5. `Notification-service`
6. `Transaction-Service`

## Example Usage

### Create a user

```bash
curl -X POST "http://localhost:8081/user/addUpdate" \
  -H "Content-Type: application/json" \
  -d '{
    "name":"Rahul",
    "contact":"9999999999",
    "email":"rahul@example.com",
    "address":"Bangalore",
    "dob":"1998-10-21",
    "userIdentifier":"PAN",
    "userIdentifierValue":"ABCDE1234F",
    "password":"secret123"
  }'
```

### Initiate a transaction

```bash
curl -X POST "http://localhost:5051/txn/initTxn?receiver=8888888888&purpose=rent&amount=20" \
  -u 9999999999:secret123
```



## License

This repository includes a `LICENSE` file at the root. Review that file for project licensing terms.
