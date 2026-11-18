# MediCare — Microservices Backend (Spring Boot & Spring Cloud)

[![CI](https://github.com/Majidied/MediCare-backend/actions/workflows/ci.yml/badge.svg)](https://github.com/Majidied/MediCare-backend/actions/workflows/ci.yml)
![Java](https://img.shields.io/badge/Java-21-ED8B00?logo=openjdk&logoColor=white)
![Spring Boot](https://img.shields.io/badge/Spring%20Boot-4.0-6DB33F?logo=springboot&logoColor=white)
![Spring Cloud](https://img.shields.io/badge/Spring%20Cloud-2025.1-6DB33F?logo=spring&logoColor=white)
![Apache Kafka](https://img.shields.io/badge/Apache%20Kafka-KRaft-231F20?logo=apachekafka&logoColor=white)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-16-4169E1?logo=postgresql&logoColor=white)
![Keycloak](https://img.shields.io/badge/Keycloak-OIDC-4D4D4D?logo=keycloak&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-Compose-2496ED?logo=docker&logoColor=white)
![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)

Event-driven microservices backend for **MediCare** (originally developed as *CabinetX*), a multi-clinic medical management platform built with **Java 21**, **Spring Boot**, **Spring Cloud** (Eureka Discovery, Gateway WebFlux, Config Server), **PostgreSQL** (database-per-service), **Apache Kafka**, **Keycloak OIDC**, **GraphQL**, **Stripe**, **STOMP WebSockets**, and **Spring AI (Google Gemini)**.

---

## Project Status

**Active / Feature-Complete MVP** — All 17 microservices, Docker Compose orchestration (with integrated Kafka & PostgreSQL), Jenkins + SonarQube pipeline, and GitHub Actions CI are configured.

---

## System Architecture

![MediCare Microservices Architecture](docs/images/architecture.svg)

```mermaid
flowchart TB
    Client["Next.js Frontend (:3000)"] -->|"REST / GraphQL / STOMP WS"| GW["gateway-service (:8080)<br/>Spring Cloud Gateway + OAuth2 JWT"]
    GW <-->|"Service Registry"| Eureka["discovery-service (:8761)<br/>Netflix Eureka"]
    KC["Keycloak OIDC (:8180)"] -.->|"JWT / JWKS / RBAC"| GW
    CFG["configuration-service (:8888)<br/>Spring Cloud Config"] -.->|"Centralized Config"| Services

    subgraph Services["14 Domain, Clinical, Billing & AI Microservices"]
        Core["Core Domain<br/>patient (:8081) • appointment (:8082)<br/>user (:8085) • clinic (:8086) • queue (:8094)"]
        Clinical["Clinical<br/>consultation (:8088, REST+GraphQL)<br/>medical-record (:8089) • medication (:8090) • prescription (:8093)"]
        Biz["Billing, Comms & AI<br/>billing (:8087) • payment (:8092)<br/>notification (:8091) • analytics (:8095) • chatbot (:8096)"]
    end

    GW --> Core & Clinical & Biz
    Services --> PG[("PostgreSQL 16 (:5432)<br/>14 isolated databases")]
    Services <--> Kafka(("Apache Kafka (:9092)<br/>Event Streaming"))
    Biz --> Stripe["Stripe API & Webhooks"]
    Biz --> Gemini["Google Gemini (Spring AI)"]
    Biz -->|"STOMP /topic/doctor/{id}/notifications"| Client
```

---

## Event-Driven Workflow: Consultation → Billing → Payment → Notification

```mermaid
sequenceDiagram
    autonumber
    participant Doc as Doctor / Frontend
    participant Cons as consultation-service
    participant Kafka as Apache Kafka
    participant Bill as billing-service
    participant Pay as payment-service
    participant Stripe as Stripe Checkout
    participant Notif as notification-service

    Doc->>Cons: Complete consultation
    Cons->>Kafka: Publish event to topic "consultation-completed"<br/>(ConsultationCompletedEventPublisher)
    Kafka-->>Bill: Consume "consultation-completed"<br/>(ConsultationCompletedEventConsumer)
    Bill->>Bill: Generate invoice (POST /api/billing/invoices)
    Bill->>Notif: POST /api/notifications/billing-ready
    Notif-->>Doc: Push STOMP alert (/topic/doctor/{userId}/notifications)

    Doc->>Pay: POST /api/payment/billing/checkout-session
    Pay->>Stripe: Create Stripe Checkout Session
    Stripe-->>Doc: Redirect to Checkout URL
    Stripe->>Pay: POST /api/payment/billing/webhook (Stripe-Signature)
    Pay->>Pay: Verify signature & update subscription/payment status
```

---

## Microservices & Port Map

| Service | Host Port | Gateway Route | Responsibility |
| :--- | :---: | :--- | :--- |
| `discovery-service` | `8761` | — | Netflix Eureka service registry & dashboard |
| `configuration-service` | `8888` | — | Spring Cloud Config Server |
| `gateway-service` | `8080` | `/*` | Reactive API Gateway + Keycloak JWT validation |
| `patient-service` | `8081` | `/api/patients/**` | Patient profiles & Kafka patient events |
| `appointment-service` | `8082` | `/api/appointments/**`, `/api/queue/**` | Scheduling, slots & waiting room queue |
| `user-service` | `8085` | `/api/users/**` | Doctor/secretary accounts & Keycloak Admin sync |
| `clinic-service` | `8086` | `/api/clinics/**` | Multi-clinic configuration & subscription plans |
| `billing-service` | `8087` | `/api/billing/**` | Automated invoicing & PDF generation |
| `consultation-service` | `8088` | `/api/consultations/**` | Clinical consultations (REST + GraphQL) |
| `medical-record-service` | `8089` | `/api/records/**` | Electronic health records (EHR) |
| `medication-service` | `8090` | `/api/medications/**` | Medication catalog & dosage reference |
| `notification-service` | `8091` | `/api/notifications/**` | Real-time STOMP WebSocket notifications |
| `payment-service` | `8092` | `/payments/**`, `/api/payment/billing/**` | Stripe Checkout & webhook verification |
| `prescription-service` | `8093` | `/api/prescriptions/**` | Digital prescriptions & signature support |
| `queue-service` | `8094` | — | Dedicated waiting queue service module |
| `analytics-service` | `8095` | `/api/statistics/**` | Clinic & global KPIs |
| `chatbot-service` | `8096` | `/chatbot/**` | Spring AI (Google Gemini) assistant |
| `postgres` | `5432` | — | PostgreSQL 16 (14 isolated DBs via `initdb`) |
| `broker` (Kafka) | `9092` | — | Apache Kafka KRaft event broker |

---

## Screenshots

### Eureka Service Discovery Dashboard (`:8761`)
![Eureka Dashboard](docs/images/eureka-dashboard.png)

> **Note:** OpenAPI/Swagger UI (`springdoc-openapi-ui`) is included in every domain microservice (`http://localhost:<service-port>/swagger-ui.html`) when the full database and config stack are running.

---

## Quick Start (Docker Compose)

1. **Clone the repository:**
   ```bash
   git clone https://github.com/Majidied/MediCare-backend.git
   cd MediCare-backend
   ```

2. **Configure environment variables:**
   ```bash
   cp .env.example .env
   ```
   Fill in your PostgreSQL passwords, Keycloak OIDC issuer/client secret, Stripe keys, and Gemini API key. Place service-specific overrides in `./env/<service-name>.env` if needed.

3. **Start the stack (PostgreSQL, Kafka, Config Server, Eureka, Gateway & Microservices):**
   ```bash
   docker compose up -d --build
   ```

4. **Verify infrastructure:**
   - **Eureka Dashboard:** `http://localhost:8761`
   - **API Gateway:** `http://localhost:8080`
   - **PostgreSQL:** `localhost:5432`
   - **Kafka Broker:** `localhost:9092`

For local JVM execution without containers, use [start-services.sh](start-services.sh) (Linux/macOS) or [start-services.ps1](start-services.ps1) (Windows) with memory profiles tuned in [.env.memory](.env.memory).

---

## Testing & CI/CD

Run unit tests inside any microservice module:

```bash
cd appointment-service
mvn -B test
```

- **GitHub Actions:** [.github/workflows/ci.yml](.github/workflows/ci.yml) builds and tests services on push/PR to `main`.
- **Jenkins + SonarQube:** [Jenkinsfile](Jenkinsfile) orchestrates sequential low-memory builds (`-Xmx384m`) and per-service SonarQube scans.

---

## License

This project is licensed under the MIT License — see [LICENSE](LICENSE) for details.
