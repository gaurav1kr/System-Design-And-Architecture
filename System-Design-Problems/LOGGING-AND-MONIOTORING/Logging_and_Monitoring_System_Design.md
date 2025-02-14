# Logging and Monitoring System Design

## **High-Level Design (HLD)**

### **Overview**
The goal is to design a Logging and Monitoring system that:
- Logs events (errors, warnings, info).
- Monitors system metrics (CPU usage, memory, disk space, etc.).
- Supports real-time querying and notifications.
- Is scalable, fault-tolerant, and multithreaded for high throughput.

### **Components**
1. **Input Layer**:
   - APIs to receive logs and metrics (RESTful endpoints or gRPC).
2. **Processing Layer**:
   - Log Parsers: Parse incoming log/metric data into structured formats.
   - Monitoring Engine: Checks metrics against defined thresholds.
   - Thread Pool: Ensures concurrent log handling.
3. **Storage Layer**:
   - **Database**: A time-series database (e.g., InfluxDB, or SQLite for simplicity).
   - Stores logs and metrics in structured format.
4. **Output Layer**:
   - Query API: Exposes logs and metrics via RESTful APIs.
   - Notification Service: Sends alerts based on defined conditions (e.g., email or webhook).

### **Data Flow**
1. **Logs** and **metrics** are pushed into the system via APIs.
2. The **Processing Layer** parses and analyzes the data.
3. Data is stored in a **database** for persistence.
4. Alerts and real-time querying are supported through the **Output Layer**.

---

## **Low-Level Design (LLD)**

### **Core Components**

#### **Thread Pool**
- Manages multiple worker threads to handle concurrent logging and monitoring requests efficiently.
- Prevents the overhead of creating and destroying threads dynamically.

#### **Database Management**
- SQLite is used as a storage layer for simplicity.
- Two tables: one for logs (`logs`) and one for metrics (`metrics`).

#### **Log and Metric Handlers**
- Separate handlers for logging events and monitoring metrics.
- Each handler interacts with the database to persist data.

#### **Monitoring Engine**
- Checks metrics (like CPU/memory usage) against predefined thresholds.
- Sends alerts (e.g., via email or console logs) when a threshold is breached.

#### **RESTful API Endpoints**
- Provide POST endpoints to accept logs and metrics.
- A GET endpoint to query logs/metrics based on filters.

---

### **Database Schema**

- **Logs Table**  
  Stores log events with their severity level and timestamp.
  ```sql
  CREATE TABLE logs (
      id INTEGER PRIMARY KEY AUTOINCREMENT,
      timestamp DATETIME DEFAULT CURRENT_TIMESTAMP,
      level TEXT NOT NULL, -- 'info', 'warning', 'error'
      message TEXT NOT NULL
  );
  ```

- **Metrics Table**  
  Stores system metrics with their names and values.
  ```sql
  CREATE TABLE metrics (
      id INTEGER PRIMARY KEY AUTOINCREMENT,
      timestamp DATETIME DEFAULT CURRENT_TIMESTAMP,
      metric_name TEXT NOT NULL, -- 'cpu_usage', 'memory_usage', etc.
      value REAL NOT NULL -- Metric value
  );
  ```

---

### **API Design**

#### **POST /log**
- Accepts log data and stores it in the database.
- **Input**:  
  ```json
  {
      "level": "error",
      "message": "Disk space is low"
  }
  ```
- **Processing**:
  - Validate input (level must be one of `info`, `warning`, `error`).
  - Insert log into the `logs` table.
- **Response**:
  ```json
  {
      "status": "success",
      "message": "Log stored successfully"
  }
  ```

#### **POST /metrics**
- Accepts a system metric and its value.
- **Input**:  
  ```json
  {
      "metric_name": "cpu_usage",
      "value": 85.5
  }
  ```
- **Processing**:
  - Validate input (metric_name must be non-empty, value must be numeric).
  - Insert metric into the `metrics` table.
  - Trigger the **Monitoring Engine** to check thresholds.
- **Response**:
  ```json
  {
      "status": "success",
      "message": "Metric stored successfully"
  }
  ```

#### **GET /query**
- Retrieves logs or metrics based on filters.
- **Input** (query params):
  ```
  type=log&level=error&from=2025-02-01&to=2025-02-14
  ```
- **Processing**:
  - Query the database for logs or metrics within the given time range.
- **Response**:
  ```json
  {
      "status": "success",
      "data": [
          {
              "timestamp": "2025-02-13 12:45:00",
              "level": "error",
              "message": "Disk space is low"
          }
      ]
  }
  ```

---

### **Monitoring Engine**

The **Monitoring Engine** continuously checks metrics against predefined thresholds.

- **Threshold Configuration** (JSON or config file):  
  ```json
  {
      "cpu_usage": 80.0,
      "memory_usage": 90.0
  }
  ```

- **Behavior**:
  1. When a new metric is inserted, the engine checks the value against the threshold.
  2. If the value exceeds the threshold, an alert is generated.

---

### **Thread Pool**

The thread pool manages concurrent requests for logging and metrics handling.

- **Workflow**:
  - Threads fetch tasks from a shared queue.
  - Tasks include logging, metrics storage, and alert generation.
  - A worker thread processes a task and returns to the pool for the next task.

---

### **Class Design**

#### **ThreadPool**
Manages multiple worker threads.
```cpp
class ThreadPool {
public:
    ThreadPool(size_t threads);      // Initialize the thread pool
    ~ThreadPool();                   // Clean up resources
    template <class F>
    void enqueue(F&& f);             // Add a task to the queue

private:
    std::vector<std::thread> workers;
    std::queue<std::function<void()>> tasks;
    std::mutex queueMutex;
    std::condition_variable condition;
    bool stop;
};
```

#### **Database**
Handles database connections and queries.
```cpp
class Database {
public:
    Database(const std::string& dbFile);  // Initialize SQLite database
    ~Database();                          // Close database connection
    void executeQuery(const std::string& query); // Execute SQL queries
};
```

#### **MonitoringEngine**
Checks metrics against thresholds and triggers alerts.
```cpp
class MonitoringEngine {
public:
    MonitoringEngine(const std::map<std::string, double>& thresholds); // Initialize thresholds
    void checkMetric(const std::string& metricName, double value);     // Check and trigger alerts

private:
    std::map<std::string, double> thresholds;
};
```

---

## **Example Flow**

### Log Ingestion:
1. User sends a `POST /log` request.
2. API handler validates the request.
3. A worker thread inserts the log into the `logs` table.

### Metric Ingestion:
1. User sends a `POST /metrics` request.
2. API handler validates the request.
3. A worker thread inserts the metric into the `metrics` table.
4. The Monitoring Engine checks if the metric exceeds its threshold.

### Query Logs/Metrics:
1. User sends a `GET /query` request.
2. API handler queries the database based on the request filters.
3. Results are returned as a JSON response.

---
