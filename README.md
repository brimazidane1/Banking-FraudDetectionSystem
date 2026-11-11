# Digital Banking System -- Fraud Detection & Payment Processing

A personal backend learning project focused on building a digital
banking system with **Spring Boot microservices**. The project explores
fund transfers, real-time fraud detection, OTP verification, distributed
transaction compensation, and payment processing.

I use this project to practice how multiple backend services work
together, how they exchange events, and how a transaction can be handled
safely when verification or an external service fails.

> **Project type:** Personal learning project\
> **Learning reference:** [Full Project: How Banks Detect Fraud \|
> Digital Banking System \| Spring Boot + Kafka + Redis +
> SAGA](https://www.youtube.com/watch?v=7Upc4iCsSh4) by Yeshendra
> Dhaker.

This is an educational project. It is not a real banking platform and
must not be used to process real funds or sensitive customer
information.

## Table of Contents

-   [Project Goals](#project-goals)
-   [Main Features](#main-features)
-   [Fraud Detection Rules](#fraud-detection-rules)
-   [Transaction and OTP Flow](#transaction-and-otp-flow)
-   [Idempotency Guard](#idempotency-guard)
-   [Saga Compensation and Refund](#saga-compensation-and-refund)
-   [Razorpay Payment Flow](#razorpay-payment-flow)
-   [Webhook Handling](#webhook-handling)
-   [Notification Flow](#notification-flow)
-   [Architecture Overview](#architecture-overview)
-   [Technology Stack](#technology-stack)
-   [Getting Started](#getting-started)
-   [Configuration](#configuration)
-   [Testing Scenarios](#testing-scenarios)
-   [Security and Reliability Notes](#security-and-reliability-notes)
-   [Possible Improvements](#possible-improvements)
-   [Reference](#reference)

## Project Goals

The main goal is to understand and practice backend development for a
banking-style application, especially the parts that involve multiple
services and transaction states.

The project focuses on:

-   Transferring funds between accounts.
-   Checking transactions for suspicious patterns in real time.
-   Asking users to verify suspicious transactions through OTP.
-   Preventing the same transaction from being processed more than once.
-   Reversing transaction effects when verification or a downstream
    operation fails.
-   Creating Razorpay payment orders and receiving payment status
    through webhooks.
-   Sending notifications based on transaction and payment events.

## Main Features

  -----------------------------------------------------------------------
  Feature                             Description
  ----------------------------------- -----------------------------------
  Fund Transfer                       Handles transfer requests between
                                      sender and receiver accounts.

  Real-Time Fraud Detection           Evaluates transaction behavior
                                      using velocity, average transaction
                                      amount, and balance-based checks.

  OTP Generation and Verification     Requests additional verification
                                      when a transaction is considered
                                      suspicious.

  Transaction State Guard             Uses the `PROCESSING` state to
                                      prevent duplicate processing while
                                      a transaction is being handled or
                                      awaiting verification.

  Refund / Compensation               Uses Saga-style compensating
                                      actions to reverse balance changes
                                      when a transaction cannot be
                                      completed.

  Razorpay Payment Order              Creates a payment order and returns
                                      the order details for the frontend
                                      checkout flow.

  Payment Webhook                     Receives payment status events from
                                      Razorpay through an HTTP POST
                                      callback.

  Event-Based Notifications           Notifies relevant users when a
                                      transfer completes, fraud is
                                      detected, or a payment integration
                                      fails.
  -----------------------------------------------------------------------

## Fraud Detection Rules

The Fraud Detection flow checks transaction behavior against three
patterns. These checks are intended to identify unusual activity and
trigger additional verification when necessary.

### 1. Velocity Check

Checks how many transactions a user performs within a defined time
window.

Fraudulent activity may involve scripts that submit many transactions in
a short period rather than a person manually initiating each
transaction.

**Example:**

-   A user makes 5, 10, or 20 transactions within one minute.
-   If the transaction count reaches the configured threshold, the
    activity is treated as suspicious.

The actual threshold should be configurable based on the application's
fraud rules.

### 2. Average Transaction Check

Compares the current transaction amount with the user's average
transaction amount.

**Example:**

-   A user's usual transaction amount is relatively small.
-   A new transaction is 3x, 4x, or 6x higher than that user's average.
-   If the amount exceeds the configured multiplier, the transaction is
    treated as suspicious.

This check helps identify transactions that are unusual for an
individual user, even when the amount might be normal for another user.

### 3. Balance Check

Checks how much of the user's available balance is being transferred.

**Example:**

-   A user attempts to transfer an amount equal to or greater than 90%
    of their available balance.
-   If the transaction crosses the configured percentage threshold, it
    is treated as suspicious.

### Fraud Detection Result

When a transaction matches one of the configured suspicious patterns,
the system can require additional user verification through OTP before
the transfer proceeds.

The fraud check is intended to add a verification step rather than
automatically treating every unusual transaction as confirmed fraud.

## Transaction and OTP Flow

The high-level flow for a transfer that requires additional verification
is:

1.  The user submits a transfer request.
2.  The application validates the request and evaluates the transaction
    using the fraud detection rules.
3.  If the transaction is considered suspicious, the transaction is kept
    in the `PROCESSING` state.
4.  The system generates an OTP and stores the temporary verification
    data in Redis with an expiration time.
5.  The user receives the OTP through the configured notification flow.
6.  The user submits the OTP for verification.
7.  If the OTP is valid and has not expired, the transaction can
    continue.
8.  If verification fails or expires, the application runs the
    configured compensation/refund flow.

### OTP Verification Cases

  -----------------------------------------------------------------------
  Case                                Expected handling
  ----------------------------------- -----------------------------------
  OTP is valid                        Verification succeeds and the
                                      transaction can continue to the
                                      next processing step.

  OTP has expired or is missing from  Treat the OTP as expired, run
  Redis (`null`)                      transaction compensation/refund,
                                      and mark the transaction as
                                      `FLAGGED`.

  OTP does not match the user's input Apply the configured invalid-OTP
                                      handling: block the account and
                                      trigger the refund/compensation
                                      flow.
  -----------------------------------------------------------------------

Redis expiration is used to limit how long an OTP remains valid. If the
OTP key is no longer available in Redis, the verification flow treats it
as expired.

## Idempotency Guard

A transaction must not credit the receiver or debit/credit an account
more than once because of repeated requests, duplicate messages, or
repeated OTP submissions.

For the verification-required flow, the transaction uses the
`PROCESSING` status as a state guard.

The intended behavior is:

1.  Set the transaction status to `PROCESSING` while it is being handled
    or waiting for OTP verification.
2.  Before applying balance changes, check that the transaction is still
    in a state that allows processing.
3.  If the same verification request is submitted again, do not repeat
    the balance operation when the transaction has already moved past
    the eligible state.
4.  Only allow the transaction to proceed through the expected state
    transition.

This status check helps prevent duplicate processing in the application
flow. For a production-grade distributed system, it should be reinforced
with atomic state updates, unique idempotency keys, and idempotent event
consumers.

## Saga Compensation and Refund

A fund transfer can involve balance changes across multiple services. If
a later step fails, the application needs to reverse the effects of
earlier successful steps.

This project uses the **Saga pattern** and compensating actions for the
refund flow.

For example, when a transfer needs to be reversed:

1.  Credit the refunded amount back to the sender.
2.  Reverse the amount previously credited to the receiver by deducting
    it from the receiver's balance.
3.  Update the transaction state according to the failure or
    fraud-handling flow.

The compensation flow is important because a distributed transaction
cannot always be rolled back with one database rollback across all
services.

The compensation operation should itself be protected against duplicate
execution so that a retry does not refund the same transaction more than
once.

## Razorpay Payment Flow

The project also explores creating a payment order through Razorpay and
completing the payment through its Checkout flow.

### Create Razorpay Payment Order

The order creation flow is:

1.  The backend creates an order in Razorpay.
2.  The backend saves the payment record in its database.
3.  The backend returns the order details to the frontend.
4.  The frontend displays the Razorpay Checkout page using the returned
    order details.
5.  The user completes the payment.
6.  Razorpay sends the payment event to the backend through a webhook.

### Payment Flow Overview

``` mermaid
sequenceDiagram
    participant User
    participant Frontend
    participant Backend as Payment Service
    participant DB as Payment Database
    participant Razorpay

    User->>Frontend: Start payment
    Frontend->>Backend: Request payment order
    Backend->>Razorpay: Create order
    Razorpay-->>Backend: Return order details
    Backend->>DB: Save payment record
    Backend-->>Frontend: Return order details
    Frontend->>User: Display Razorpay Checkout
    User->>Razorpay: Complete payment
    Razorpay->>Backend: Send payment webhook
    Backend->>DB: Update payment status
```

The payment record is stored before the frontend receives the order
details, so the backend has a record associated with the payment flow.

## Webhook Handling

A webhook allows Razorpay to notify the backend when a payment-related
event occurs.

After the user completes the Checkout flow:

1.  Razorpay sends an HTTP POST request to the configured backend
    webhook endpoint.
2.  The backend receives the event payload and identifies the payment
    event/status.
3.  The backend processes the event and updates the corresponding
    payment record.
4.  The application triggers the appropriate follow-up action, such as
    updating the transaction flow or notifying the user.

Webhook handling should account for repeated delivery. The backend
should verify the webhook signature and process each event idempotently
before relying on it for payment state changes.

## Notification Flow

Notifications are triggered by important transaction and payment
outcomes.

  Event                                    Notification recipient
  ---------------------------------------- ------------------------
  Transaction completed                    Sender and receiver
  Fraud detected / verification required   Relevant user
  External payment service failure         Relevant user

The Notification Service can consume application events and send the
configured notification to the appropriate user.

## Architecture Overview

The following diagram shows the main logical components and interactions
covered by this project. It is a conceptual overview; refer to the
actual service code and configuration for the exact implementation.

``` mermaid
flowchart TD
    Client[Frontend / Client] --> Gateway[API Gateway]
    Gateway --> Transaction[Transaction Flow]
    Transaction --> Fraud[Fraud Detection]

    Fraud -->|Suspicious transaction| OTP[OTP Verification]
    OTP <--> Redis[(Redis - OTP with TTL)]
    OTP -->|Verified| Transaction
    OTP -->|Expired or invalid| Saga[Saga Compensation]

    Transaction --> Account[Account / Balance Operations]
    Saga -->|Credit sender and reverse receiver credit| Account

    Transaction --> Kafka[(Apache Kafka)]
    Kafka --> Notification[Notification Flow]

    Client --> Payment[Payment Flow]
    Payment <--> Razorpay[Razorpay]
    Razorpay -->|HTTP POST webhook| Payment
    Payment --> PaymentDB[(Payment Database)]
    Payment --> Kafka
```

### Communication and Infrastructure

-   **Spring Boot microservices:** Separate backend responsibilities
    into services.
-   **Apache Kafka:** Supports asynchronous event communication between
    application components.
-   **Redis:** Stores temporary OTP data and supports expiration through
    TTL.
-   **API Gateway:** Acts as the entry point for client requests and
    routes them to backend services.
-   **Razorpay:** Provides the payment order and Checkout integration,
    with payment updates delivered through webhooks.
-   **Relational database:** Persists transaction, account, and payment
    records according to each service's configuration.
-   **Docker Compose:** Helps run the application services and
    supporting infrastructure in a local containerized environment.

## Technology Stack

  -----------------------------------------------------------------------
  Technology                          Usage
  ----------------------------------- -----------------------------------
  Java                                Main programming language

  Spring Boot                         Backend application framework

  Spring Data JPA / Hibernate         Database access and ORM, where
                                      configured

  Apache Kafka                        Asynchronous event messaging

  Redis                               Temporary OTP storage, expiration,
                                      and caching where configured

  Microservices                       Separating backend business
                                      responsibilities

  API Gateway                         Request routing

  Saga Pattern                        Distributed transaction
                                      compensation

  Razorpay                            Payment order and Checkout
                                      integration

  Docker / Docker Compose             Containerizing and running services
                                      locally

  Relational Database                 Persistent application data
  -----------------------------------------------------------------------

Use the actual versions and database products defined in the
repository's build files and configuration.

## Getting Started

### Prerequisites

Install the tools required by the repository:

-   JDK version specified by the project.
-   Maven, or the Maven Wrapper included in the repository.
-   Docker Engine or Docker Desktop.
-   Docker Compose.
-   Git.

### 1. Clone the repository

``` bash
git clone <your-repository-url>
cd <repository-folder>
```

### 2. Configure the services

Review each service's configuration and provide the required database,
Kafka, Redis, and Razorpay settings.

Do not commit real credentials or secrets to source control.

### 3. Start the required containers

If the repository includes a Docker Compose file at the project root,
start the configured services with:

``` bash
docker compose up -d
```

Check container status:

``` bash
docker compose ps
```

### 4. Run the Spring Boot services

Use the Maven Wrapper or the run configuration provided by the project.
For example:

``` bash
./mvnw spring-boot:run
```

On Windows:

``` bat
mvnw.cmd spring-boot:run
```

Run the command from the relevant service directory. The exact startup
order and commands depend on the repository configuration.

### 5. Test the APIs

Use Postman or another API client to test the endpoints exposed by the
API Gateway. Refer to the controllers and application configuration for
the actual routes, ports, and request payloads.

### 6. Stop the containers

``` bash
docker compose down
```

Review the Compose configuration before removing volumes, because
volumes may contain local database or application data.

## Configuration

The exact property names depend on the implementation. Review
configuration for:

-   **Database:** connection URL, username, password, and
    schema/database name.
-   **Kafka:** bootstrap server, topic names, producer, and consumer
    settings.
-   **Redis:** host, port, and OTP expiration/TTL settings.
-   **API Gateway:** service routes and gateway settings.
-   **Razorpay:** API credentials, webhook endpoint, and webhook secret.
-   **Notifications:** provider configuration and credentials, if an
    external notification provider is used.

Keep credentials outside source control. Use environment variables or
local configuration files excluded from Git. Never commit API keys,
passwords, access tokens, private keys, or webhook secrets.

## Testing Scenarios

  -----------------------------------------------------------------------
  Scenario                            What to verify
  ----------------------------------- -----------------------------------
  Normal fund transfer                The transfer follows the expected
                                      flow and sender/receiver balances
                                      are updated correctly.

  Velocity threshold reached          Multiple transactions within the
                                      configured time window trigger
                                      suspicious-activity handling.

  Average transaction threshold       A transaction that is unusually
  reached                             large compared with the user's
                                      average triggers the configured
                                      fraud response.

  Balance threshold reached           A transfer reaching the configured
                                      percentage of the user's balance
                                      triggers the configured fraud
                                      response.

  Valid OTP                           Verification succeeds and the
                                      transaction continues.

  OTP expired in Redis                The OTP is treated as expired,
                                      compensation/refund is triggered,
                                      and the transaction is marked
                                      `FLAGGED`.

  Invalid OTP                         The configured invalid-OTP handling
                                      blocks the account and triggers
                                      compensation/refund.

  Repeated OTP submission             The transaction state guard
                                      prevents the same balance operation
                                      from being applied twice.

  Compensation                        The sender is credited back and the
                                      receiver's previous credit is
                                      reversed as intended.

  Razorpay order creation             An order is created and its details
                                      are saved and returned to the
                                      frontend.

  Payment webhook                     The backend receives the HTTP POST
                                      event and updates the corresponding
                                      payment status.

  Transaction completed               Sender and receiver receive the
                                      configured notifications.

  Fraud detected                      The relevant user receives a
                                      fraud/verification notification.

  Payment service failure             The relevant user receives a
                                      failure notification.
  -----------------------------------------------------------------------

## Security and Reliability Notes

This is an educational project, not production-ready banking software.

Before using a system like this in a real financial environment,
additional controls would be required, including:

-   Strong authentication and authorization across services.
-   Secure storage and rotation of API credentials and secrets.
-   OTP attempt limits, secure OTP generation, and protection against
    OTP replay.
-   Atomic transaction-state transitions and durable idempotency
    controls.
-   Idempotent Kafka consumers and safe handling of duplicate events.
-   Razorpay webhook signature verification and replay protection.
-   Reliable retry handling and dead-letter processing for failed
    events.
-   Audit logs, monitoring, centralized logging, and distributed
    tracing.
-   Reconciliation and operational procedures for incomplete or failed
    compensation.
-   Security, performance, and compliance testing.

Use dummy accounts and test payment credentials only. Do not connect
this learning project to real customer accounts or real funds.

## Possible Improvements

-   Add unit and integration tests using JUnit and Mockito.
-   Add explicit idempotency keys and atomic state transitions for
    transfer and refund operations.
-   Add Kafka retry policies and dead-letter topics.
-   Add webhook signature verification and duplicate-event protection.
-   Add monitoring, centralized logging, and distributed tracing.
-   Automate build and deployment with Jenkins or GitHub Actions.
-   Practice deploying the containerized services to AWS EC2 and storing
    images in Amazon ECR.

## Reference

-   [Full Project: How Banks Detect Fraud \| Digital Banking System \|
    Spring Boot + Kafka + Redis +
    SAGA](https://www.youtube.com/watch?v=7Upc4iCsSh4)
-   Creator: [Yeshendra
    Dhaker](https://www.youtube.com/@YeshendraDhaker)

------------------------------------------------------------------------

*This README reflects the project flow and business rules described in
my learning notes. Update service names, API routes, ports, database
products, configuration keys, and implementation details to match the
code in this repository.*
