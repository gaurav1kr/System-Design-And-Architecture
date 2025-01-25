
# High-Level Design (HLD) and Low-Level Design (LLD)

## High-Level Design (HLD):
- **Overview:** HLD is the *big picture* of the system. It describes the system architecture, components, and how they interact without going into details of implementation.
- **Focus Areas:**
  - System architecture (e.g., client-server, microservices)
  - Main components/modules of the system
  - Data flow and control flow across modules
  - External interfaces (APIs, services)
  - Technology stack (languages, frameworks, databases)
- **Audience:** Solution architects, senior developers, technical leads
- **Documentation:** Often includes block diagrams, architecture diagrams, flowcharts, etc.

## Low-Level Design (LLD):
- **Overview:** LLD goes deeper into the details of the system. It provides a more granular view of each component/module described in HLD, focusing on how they are implemented.
- **Focus Areas:**
  - Detailed class diagrams, method definitions, attributes
  - Data structures, algorithms, and logic flow
  - Database schema design and queries
  - Pseudo code or detailed specifications for each component
- **Audience:** Developers and engineers who will implement the code
- **Documentation:** Class diagrams, sequence diagrams, pseudo code, function specifications

---

## Example: A Simple Online Banking System

### High-Level Design (HLD):
Imagine a system for an online banking application. The HLD will describe:

- **Architecture:**
  - Client-server architecture: Mobile/desktop clients, RESTful APIs, backend services, database
- **Components:**
  - **User Interface (UI):** Login, transaction history, transfer funds
  - **Authentication Service:** Validates users
  - **Transaction Service:** Handles fund transfers, payments
  - **Database:** Stores account details, transactions
- **Data Flow:**
  - A user requests to transfer funds → API receives request → Authentication service validates the user → Transaction service processes the request → Database updates the records
- **Technology Stack:**
  - Frontend: React.js
  - Backend: Node.js with Express.js
  - Database: PostgreSQL
  - External APIs: Payment gateways, SMS services for OTP

This would be represented as an architecture diagram with blocks showing UI, services, and database interactions.

### Low-Level Design (LLD):
For the **Transaction Service**, LLD will provide detailed design:

- **Classes and Methods:**
  - Class: `TransactionService`
    - Method: `transferFunds(fromAccount, toAccount, amount)`
      - Validates sufficient balance
      - Deducts amount from `fromAccount`
      - Adds amount to `toAccount`
      - Logs transaction details
- **Database Schema:**
  - Table: `accounts`
    - Fields: `account_id`, `balance`, `user_id`
  - Table: `transactions`
    - Fields: `transaction_id`, `from_account`, `to_account`, `amount`, `timestamp`
- **Sequence Diagram:** Illustrating the interaction between the API, transaction service, authentication service, and the database during fund transfer.
