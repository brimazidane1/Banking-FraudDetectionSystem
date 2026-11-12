# Digital Banking System --- Fraud Detection & Payment Processing

A personal backend learning project that explores how a digital banking
system can be designed with **Spring Boot microservices**, **Apache
Kafka**, **Redis**, and distributed transaction patterns.

The project focuses on fund transfers, rule-based fraud screening, OTP
verification, compensation/refund flows, and payment processing through
Razorpay.

> **Learning project only.** This repository is for educational
> purposes. It is not a production banking platform and must not be used
> to process real funds or store real customer information.

## Table of Contents

-   [Project Goals](#project-goals)
-   [Main Features](#main-features)
-   [Architecture Overview](#architecture-overview)
-   [Fraud Detection Rules](#fraud-detection-rules)
-   [Transaction and OTP Flow](#transaction-and-otp-flow)
-   [Kafka Event Mapping](#kafka-event-mapping)
-   [Idempotency Guard](#idempotency-guard)
-   [Saga Compensation and Refund](#saga-compensation-and-refund)
-   [Razorpay Payment Flow](#razorpay-payment-flow)
-   [Webhook Handling](#webhook-handling)
-   [Notification Flow](#notification-flow)
-   [Technology Stack](#technology-stack)
-   [Getting Started](#getting-started)
-   [Configuration](#configuration)
-   [Testing Scenarios](#testing-scenarios)
-   [Security and Reliability Notes](#security-and-reliability-notes)
-   [Possible Improvements](#possible-improvements)
-   [Reference](#reference)

## Project Goals

The main goals of this project are to:

-   Build backend services with a microservices-oriented architecture
    using Spring Boot.
-   Understand asynchronous communication between services with Apache
    Kafka.
-   Implement rule-based fraud screening for fund transfers.
-   Handle OTP generation and verification within the Transaction
    Service, with Redis used for temporary OTP storage and expiration.
-   Explore distributed transaction handling with the Saga pattern and
    compensation actions.
-   Integrate an external payment provider through Razorpay order
    creation and webhook callbacks.
-   Understand how services react to business events without tightly
    coupling every operation through synchronous API calls.

## Main Features

-   **Account management:** account-related operations and balance
    updates.
-   **Fund transfers:** initiate and process transfers between accounts.
-   **Fraud screening:** evaluate transactions using configurable rule
    thresholds.
-   **OTP verification:** generate and verify OTPs in the Transaction
    Service.
-   **Temporary OTP storage:** use Redis with a time-to-live (TTL) for
    OTP data.
-   **Event-driven communication:** publish and consume business events
    through Kafka.
-   **Compensation/refund:** reverse balance changes when a transfer
    must be rolled back.
-   **Payment processing:** create Razorpay orders and process payment
    webhook events.
-   **Notifications:** react to OTP, transaction, fraud, refund, and
    payment events.

## Architecture Overview

The system is divided into services with separate responsibilities.
Kafka is used for asynchronous event communication between the
Transaction, Fraud Detection, Account, Payment, and Notification
services. Redis is used by the Transaction Service for temporary OTP
storage.

``` mermaid
flowchart TB
    Client[Frontend / Client]
    Gateway[API Gateway]

    Account[Account Service]
    Transaction[Transaction Service]
    Fraud[Fraud Detection Service]
    Payment[Payment Service]
    Notification[Notification Service]

    Kafka[(Apache Kafka)]
    Redis[(Redis)]
    Razorpay[Razorpay]

    Client --> Gateway
    Gateway --> Account
    Gateway --> Transaction
    Gateway --> Payment

    Transaction <-->|Store and retrieve OTP| Redis

    Transaction -->|transaction.initiated| Kafka
    Kafka -->|transaction.initiated| Fraud

    Fraud -->|verification.required| Kafka
    Fraud -->|fraud.check.clean| Kafka
    Kafka -->|verification.required / fraud.check.clean| Transaction

    Transaction -->|transaction.otp.generated| Kafka
    Transaction -->|transaction.completed| Kafka
    Transaction -->|fraud.detected| Kafka
    Transaction -->|transaction.refunded| Kafka

    Kafka -->|transaction.completed / fraud.detected| Account
    Kafka -->|OTP, transaction, fraud and refund events| Notification

    Payment <--> Razorpay
    Razorpay -->|Payment webhook| Payment
    Payment -->|payment.completed / payment.failed| Kafka
```

The diagram is a conceptual overview of the service and event
relationships. The exact request paths, persistence boundaries, and
deployment topology depend on the implementation in the repository.

### Service Responsibilities

  -----------------------------------------------------------------------
  Service                             Responsibility
  ----------------------------------- -----------------------------------
  **Account Service**                 Handles account-related operations
                                      and balance updates. Consumes
                                      transaction outcome events to apply
                                      the corresponding account-side
                                      changes.

  **Transaction Service**             Handles transfer requests,
                                      transaction state, OTP generation
                                      and verification, and the
                                      transaction-side coordination of
                                      completion or compensation.

  **Fraud Detection Service**         Consumes initiated transaction
                                      events, evaluates fraud rules, and
                                      publishes the screening result.

  **Payment Service**                 Creates payment orders, handles
                                      Razorpay webhook callbacks, updates
                                      payment records, and publishes
                                      payment outcome events.

  **Notification Service**            Consumes relevant Kafka events and
                                      sends notifications for OTP,
                                      transaction, fraud, refund, and
                                      payment updates.
  -----------------------------------------------------------------------

**OTP is not a separate microservice in this project.** OTP generation
and verification are handled by the Transaction Service, which uses
Redis to store temporary OTP values.

## Fraud Detection Rules

The Fraud Detection Service evaluates transactions using three rule
patterns. The thresholds below are examples of configurable rules, not
universal banking limits.

### 1. Velocity Check

Detects an unusually high number of transactions within a short period.

Examples: - 5 transactions within 1 minute. - 10 transactions within 1
minute. - 20 transactions within 1 minute.

A transaction can be flagged for additional verification when the
configured transaction-count threshold is reached.

### 2. Average Transaction Check

Compares the current transaction amount with the user's historical
average transaction amount.

Examples: - Current amount exceeds 3x the user's average. - Current
amount exceeds 4x the user's average. - Current amount exceeds 6x the
user's average.

The configured multiplier determines when the transaction requires
additional verification.

### 3. Balance Check

Checks whether the requested transfer consumes an unusually large
portion of the user's available balance.

Example: - The transfer amount is around 90% of the available balance.

The percentage is configurable and should be treated as a screening
signal, not proof of fraudulent activity.

### Screening Outcome

The rules are used to identify transactions that may require additional
verification. A suspicious screening result does not automatically prove
that fraud has occurred. The Fraud Detection Service communicates its
result to the Transaction Service through Kafka.

## Transaction and OTP Flow

The Transaction Service coordinates the transfer workflow. When a
transfer is initiated, it publishes an event so that the Fraud Detection
Service can evaluate the transaction asynchronously.

``` mermaid
sequenceDiagram
    participant User
    participant TS as Transaction Service
    participant Kafka
    participant FDS as Fraud Detection Service
    participant Redis
    participant NS as Notification Service

    User->>TS: Submit transfer request
    TS->>TS: Validate request and create transaction
    TS->>Kafka: Publish transaction.initiated
    Kafka->>FDS: Deliver transaction.initiated
    FDS->>FDS: Evaluate fraud rules

    alt Additional verification required
        FDS->>Kafka: Publish verification.required
        Kafka->>TS: Deliver verification.required
        TS->>TS: Check transaction state
        TS->>TS: Generate OTP
        TS->>Redis: Store OTP with TTL
        TS->>Kafka: Publish transaction.otp.generated
        Kafka->>NS: Deliver transaction.otp.generated
        NS-->>User: Send OTP notification
        User->>TS: Submit OTP
        TS->>Redis: Retrieve and validate OTP
        TS->>TS: Continue or compensate transaction
    else Screening result is clean
        FDS->>Kafka: Publish fraud.check.clean
        Kafka->>TS: Deliver fraud.check.clean
        TS->>TS: Continue normal transaction flow
    end
```

### OTP Verification Cases

When additional verification is required, the Transaction Service
generates an OTP and stores it in Redis with an expiration time.

  -----------------------------------------------------------------------
  Scenario                            Expected handling
  ----------------------------------- -----------------------------------
  **Valid OTP**                       Verification succeeds and the
                                      transaction can continue through
                                      its completion flow.

  **Expired or missing OTP**          Redis returns no usable OTP value
                                      (for example, `null` after
                                      expiration). The configured failure
                                      flow triggers compensation/refund
                                      and updates the transaction state,
                                      such as marking it `FLAGGED`.

  **Incorrect OTP**                   The configured failure flow can
                                      block the account and trigger
                                      compensation/refund.
  -----------------------------------------------------------------------

The exact state transitions and failure handling should be checked
against the implementation. Redis TTL provides automatic expiration for
temporary OTP values; it does not replace OTP attempt limits or replay
protection.

## Kafka Event Mapping

The following table documents the producer-consumer relationships used
by this project.

  -----------------------------------------------------------------------------
  Kafka Topic                   Producer                Consumer
  ----------------------------- ----------------------- -----------------------
  `transaction.initiated`       Transaction Service     Fraud Detection Service

  `verification.required`       Fraud Detection Service Transaction Service

  `fraud.check.clean`           Fraud Detection Service Transaction Service

  `transaction.otp.generated`   Transaction Service     Notification Service

  `transaction.completed`       Transaction Service     Account Service,
                                                        Notification Service

  `fraud.detected`              Transaction Service     Account Service,
                                                        Notification Service

  `transaction.refunded`        Transaction Service     Notification Service

  `payment.completed`           Payment Service         Notification Service

  `payment.failed`              Payment Service         Notification Service
  -----------------------------------------------------------------------------

### Event Flow Notes

-   `transaction.initiated` starts asynchronous fraud screening.
-   `verification.required` and `fraud.check.clean` communicate the
    screening outcome back to the Transaction Service.
-   `transaction.otp.generated` is published by the Transaction Service
    after OTP generation; the Notification Service consumes it to notify
    the user.
-   `transaction.completed` is consumed by the Account Service and
    Notification Service.
-   In this implementation, **`fraud.detected` is published by the
    Transaction Service**, then consumed by the Account Service and
    Notification Service.
-   `transaction.refunded` is consumed by the Notification Service.
-   Payment outcome events are published by the Payment Service and
    consumed by the Notification Service.

## Idempotency Guard

Kafka consumers may receive duplicate events, and clients may retry
requests. The transaction flow therefore uses the transaction status
(for example, `PROCESSING`) as a state guard before continuing certain
operations, such as handling a verification-required result.

The guard helps reduce duplicate processing, such as generating or
handling an OTP repeatedly for a transaction that is already being
processed.

However, a status check by itself is **not** a complete exactly-once
processing guarantee. In a concurrent or distributed environment,
production-grade idempotency should also consider:

-   Atomic state transitions, such as conditional database updates.
-   A unique idempotency key for client-submitted operations.
-   Idempotent Kafka consumers that safely handle repeated event
    delivery.
-   Duplicate-event tracking or an inbox pattern where appropriate.
-   Compensation operations that cannot be applied more than once.

## Saga Compensation and Refund

A distributed transfer can involve balance changes in more than one
service or database. A single database rollback cannot automatically
undo work that has already committed in another service.

When a transfer needs to be reversed, the compensation flow is
responsible for restoring the affected balances.

### Compensation Steps

1.  **Credit the sender:** return the refunded amount to the sender's
    account.
2.  **Reverse the receiver credit:** deduct the amount that was
    previously credited to the receiver.
3.  **Update transaction state:** record the appropriate failure,
    flagged, or refunded state according to the business flow.
4.  **Publish the outcome:** publish the relevant event, such as
    `transaction.refunded`, so downstream services can react.

``` mermaid
flowchart TD
    Start[Transfer requires reversal]
    Sender[Credit refunded amount to sender]
    Receiver[Reverse credit from receiver]
    State[Update transaction state]
    Event[Publish transaction.refunded]
    Done[Compensation flow completed]

    Start --> Sender
    Sender --> Receiver
    Receiver --> State
    State --> Event
    Event --> Done
```

Compensation must itself be idempotent. If the same failure event is
processed more than once, the system must not refund the sender twice or
deduct the receiver's balance repeatedly.

## Razorpay Payment Flow

The Payment Service integrates with Razorpay for payment processing.

``` mermaid
sequenceDiagram
    participant User
    participant Frontend
    participant PS as Payment Service
    participant DB as Payment Database
    participant RZ as Razorpay
    participant Kafka
    participant NS as Notification Service

    User->>Frontend: Start payment
    Frontend->>PS: Request payment order
    PS->>RZ: Create order
    RZ-->>PS: Return order details
    PS->>DB: Save payment record
    PS-->>Frontend: Return order details
    Frontend->>RZ: Open Razorpay Checkout
    User->>RZ: Complete payment
    RZ->>PS: Send payment webhook
    PS->>PS: Process payment event
    PS->>DB: Update payment record
    alt Payment successful
        PS->>Kafka: Publish payment.completed
    else Payment failed
        PS->>Kafka: Publish payment.failed
    end
    Kafka->>NS: Deliver payment outcome event
    NS-->>User: Send payment notification
```

### Payment Steps

1.  The frontend asks the Payment Service to create a payment order.
2.  The Payment Service creates an order through Razorpay.
3.  The Payment Service stores a payment record and returns the order
    details to the frontend.
4.  The frontend displays Razorpay Checkout.
5.  The user completes the payment through Razorpay.
6.  Razorpay sends an HTTP `POST` webhook to the backend.
7.  The Payment Service processes the webhook and updates the payment
    record.
8.  The Payment Service publishes `payment.completed` or
    `payment.failed`.
9.  The Notification Service consumes the event and sends the
    corresponding notification.

## Webhook Handling

Razorpay uses webhooks to notify the backend about payment events.

The Payment Service is responsible for receiving and processing the
webhook callback, updating the payment record, and publishing the
corresponding Kafka event.

For a production integration, webhook handling should include:

-   Verify the webhook signature using the configured webhook secret.
-   Validate the event and payment/order identifiers.
-   Handle duplicate webhook deliveries idempotently.
-   Reject or safely ignore events that have already been processed.
-   Avoid trusting frontend payment status as the sole source of truth.
-   Keep webhook secrets outside source control.

These are important reliability and security requirements; verify each
one against the actual implementation before describing it as already
implemented.

## Notification Flow

The Notification Service consumes Kafka events and sends the relevant
user notifications.

  -----------------------------------------------------------------------
  Kafka Topic                         Notification
  ----------------------------------- -----------------------------------
  `transaction.otp.generated`         Sends the generated OTP to the
                                      relevant user.

  `transaction.completed`             Notifies the sender and receiver
                                      about the completed transaction.

  `fraud.detected`                    Notifies the relevant user about
                                      suspicious activity.

  `transaction.refunded`              Notifies the relevant user about a
                                      refund or compensation.

  `payment.completed`                 Notifies the user that the payment
                                      is completed.

  `payment.failed`                    Notifies the user that the payment
                                      failed.
  -----------------------------------------------------------------------

## Technology Stack

  -----------------------------------------------------------------------
  Technology                          Usage
  ----------------------------------- -----------------------------------
  **Java**                            Main programming language.

  **Spring Boot**                     Backend application framework.

  **Spring Data JPA / Hibernate**     Data access and object-relational
                                      mapping.

  **Apache Kafka**                    Asynchronous event-driven
                                      communication between services.

  **Redis**                           Temporary OTP storage with
                                      expiration (TTL).

  **Microservices**                   Separation of backend business
                                      responsibilities.

  **API Gateway**                     Entry point and request routing to
                                      backend services.

  **Saga Pattern**                    Coordination of distributed
                                      transaction compensation.

  **Razorpay**                        Payment order and Checkout
                                      integration.

  **Docker / Docker Compose**         Containerization and local service
                                      orchestration.

  **Relational Database**             Persistent application data.
  -----------------------------------------------------------------------

The exact database products, framework versions, service ports, and
deployment configuration should be taken from the project configuration
files.

## Getting Started

### Prerequisites

Install the tools required by the services in this repository:

-   JDK version compatible with the project.
-   Maven or the included Maven Wrapper.
-   Docker Engine / Docker Desktop.
-   Docker Compose.
-   Git.

### Clone the Repository

``` bash
git clone <your-repository-url>
cd <repository-folder>
```

### Start Infrastructure

If the repository includes a Docker Compose configuration for
infrastructure and supporting services:

``` bash
docker compose up -d
docker compose ps
```

### Run a Spring Boot Service

Move to the relevant service directory and run it using the Maven
Wrapper:

**macOS / Linux**

``` bash
./mvnw spring-boot:run
```

**Windows**

``` bat
mvnw.cmd spring-boot:run
```

If the repository uses a root-level multi-module Maven build, follow its
module-specific run instructions instead. The commands above are generic
examples; adjust the directory and command to match the actual
repository structure.

### Stop Docker Services

``` bash
docker compose down
```

## Configuration

Configure each service using the project's application configuration
files or environment variables.

Typical configuration categories include:

-   Database connection details.
-   Kafka bootstrap servers and topic names.
-   Redis host, port, and connection settings.
-   Service-to-service URLs.
-   Razorpay API key and secret.
-   Razorpay webhook secret.
-   OTP expiration duration and related verification settings.

Do not commit real credentials, API secrets, webhook secrets, or
personal information to the repository. Use local environment variables
or an appropriate secrets-management solution.

## Testing Scenarios

The following scenarios are useful when validating the main business
flows.

  --------------------------------------------------------------------------
  Area                    Scenario                Expected result
  ----------------------- ----------------------- --------------------------
  Transfer                Submit a valid transfer Transaction is created and
                          request                 fraud screening is
                                                  initiated.

  Fraud screening         Transaction is within   Fraud Detection Service
                          configured thresholds   publishes
                                                  `fraud.check.clean`.

  Fraud screening         Transaction reaches a   Fraud Detection Service
                          configured suspicious   publishes
                          threshold               `verification.required`.

  OTP                     Submit a valid OTP      Verification succeeds and
                                                  the transaction continues.

  OTP                     Submit an expired or    The configured failure and
                          missing OTP             compensation flow is
                                                  triggered.

  OTP                     Submit an incorrect OTP The configured invalid-OTP
                                                  handling is triggered.

  Idempotency             Process the same event  Duplicate processing
                          more than once          should be prevented or
                                                  safely handled.

  Compensation            Reverse a transfer      Sender and receiver
                                                  balances are compensated
                                                  correctly.

  Payment                 Payment succeeds        Payment Service publishes
                                                  `payment.completed`.

  Payment                 Payment fails           Payment Service publishes
                                                  `payment.failed`.

  Notification            Consume a supported     The corresponding
                          business event          notification is sent.
  --------------------------------------------------------------------------

## Security and Reliability Notes

This project is intended for learning and should not be considered
production-ready.

Before using a similar design in a real financial application, consider:

-   **Authentication and authorization:** protect APIs and enforce
    access control for account and transaction operations.
-   **Sensitive data:** never use real customer information or
    production credentials in a learning environment.
-   **OTP security:** use cryptographically secure generation, short
    expiration, attempt limits, replay protection, and safe delivery.
-   **Idempotency:** protect API requests, Kafka consumers, webhook
    processing, and compensation operations against duplicate execution.
-   **Atomic state changes:** use database constraints or conditional
    updates to prevent conflicting concurrent transitions.
-   **Kafka reliability:** define retry policies, dead-letter handling,
    consumer error handling, and event schema/versioning strategies.
-   **Webhook security:** verify Razorpay webhook signatures and
    validate event identifiers before updating payment state.
-   **Observability:** add structured logs, metrics, monitoring, and
    distributed tracing.
-   **Reconciliation:** provide a way to identify and resolve incomplete
    transfers or compensation steps.
-   **Testing and compliance:** perform security, load,
    failure-recovery, and applicable regulatory testing.

## Possible Improvements

Potential next steps for this learning project:

-   Add more unit and integration tests using JUnit and Mockito.
-   Strengthen idempotency with unique request keys and atomic state
    transitions.
-   Add Kafka retry and dead-letter topic handling.
-   Add event schema validation and versioning.
-   Verify Razorpay webhook signatures and add duplicate-event
    protection.
-   Improve structured logging, monitoring, and distributed tracing.
-   Add automated build and test workflows using Jenkins or GitHub
    Actions.
-   Practice container deployment to AWS EC2 and image management
    through Amazon ECR.
-   Add reconciliation handling for transfers that stop partway through
    a compensation flow.

## Reference

This project is based on concepts explored while following the tutorial:

-   **Video:** [Full Project: How Banks Detect Fraud \| Digital Banking
    System \| Spring Boot + Kafka + Redis +
    SAGA](https://www.youtube.com/watch?v=7Upc4iCsSh4)
-   **Creator:** Yeshendra Dhaker

The repository is a personal learning implementation and may differ from
the tutorial or from real-world banking systems.
