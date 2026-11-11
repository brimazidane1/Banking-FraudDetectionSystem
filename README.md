
# Digital Banking System
### Real-Time Fraud Detection & Payment Processing

A microservices-based digital banking learning project built with **Java, Spring Boot, Apache Kafka, Redis, and Saga-based transaction compensation**.

---

## 📌 Overview

This project demonstrates a digital banking system with fund transfers, real-time fraud detection, OTP-based transaction verification, payment processing, and event-driven notifications.

The system separates business responsibilities into multiple backend services. Apache Kafka supports asynchronous communication, Redis stores temporary OTP data, and the Saga pattern coordinates compensating actions when a transaction cannot be completed.

