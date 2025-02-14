# Notification Service Design

## **🔹 High-Level Design (HLD)**
### **1. System Overview**  
A **Notification Service** handles multiple notification types, ensuring high throughput and scalability via multithreading and a thread pool.

### **2. Components**  
1. **API Gateway**: Receives notification requests.  
2. **Notification Queue**: Stores messages for processing.  
3. **Thread Pool Worker**: Processes queued notifications.  
4. **Notification Handlers**: Sends notifications via Email, SMS, Push, etc.  
5. **Database**: Stores notification history and status.  

### **3. Technologies Used**  
- **C++17/20** for modern C++ features.  
- **Multithreading** via `std::thread`.  
- **Thread Pool** for concurrent execution.  
- **Queue** (thread-safe) to manage notifications.  

## **🔹 Low-Level Design (LLD)**  
### **Class Diagram**  

```plaintext
+-----------------------+
|   NotificationService |
|-----------------------|
| - threadPool         |
| - queue              |
| - handlers           |
|-----------------------|
| + sendNotification() |
| + workerThreads()    |
+-----------------------+
           |
           |
           v
+-----------------------+
|    ThreadPool        |
|-----------------------|
| - workers            |
| - taskQueue          |
| - mutex/conditionVar |
|-----------------------|
| + enqueueTask()      |
| + workerThread()     |
+-----------------------+
           |
           |
           v
+-----------------------+
|   NotificationQueue  |
|-----------------------|
| - queue              |
| - mutex/conditionVar |
|-----------------------|
| + push()             |
| + pop()              |
+-----------------------+
           |
           |
           v
+-----------------------+
|  NotificationHandler |
|-----------------------|
| - sendEmail()        |
| - sendSMS()          |
| - sendPush()         |
+-----------------------+
```

## **🔹 API Design**  
### **1. Send Notification API**  
**Endpoint:** `/sendNotification`  
**Method:** `POST`  
**Request:**  
```json
{
  "userId": 12345,
  "type": "email",
  "message": "Welcome to our service!"
}
```
**Response:**  
```json
{
  "status": "queued",
  "notificationId": 98765
}
```

## **🔹 C++ Implementation**  

### **notification_service.cpp**
```cpp
#include <iostream>
#include <queue>
#include <vector>
#include <thread>
#include <mutex>
#include <condition_variable>
#include <functional>
#include <unordered_map>
#include <memory>

// Thread Pool Implementation
class ThreadPool {
public:
    ThreadPool(size_t numThreads);
    ~ThreadPool();
    void enqueueTask(std::function<void()> task);

private:
    std::vector<std::thread> workers;
    std::queue<std::function<void()>> tasks;
    std::mutex queueMutex;
    std::condition_variable condition;
    bool stop;
    void workerThread();
};

ThreadPool::ThreadPool(size_t numThreads) : stop(false) {
    for (size_t i = 0; i < numThreads; ++i) {
        workers.emplace_back(&ThreadPool::workerThread, this);
    }
}

ThreadPool::~ThreadPool() {
    {
        std::unique_lock<std::mutex> lock(queueMutex);
        stop = true;
    }
    condition.notify_all();
    for (std::thread &worker : workers) {
        worker.join();
    }
}

void ThreadPool::enqueueTask(std::function<void()> task) {
    {
        std::unique_lock<std::mutex> lock(queueMutex);
        tasks.push(std::move(task));
    }
    condition.notify_one();
}

void ThreadPool::workerThread() {
    while (true) {
        std::function<void()> task;
        {
            std::unique_lock<std::mutex> lock(queueMutex);
            condition.wait(lock, [this] { return stop || !tasks.empty(); });
            if (stop && tasks.empty()) return;
            task = std::move(tasks.front());
            tasks.pop();
        }
        task();
    }
}

// Notification Service
class NotificationService {
public:
    NotificationService(size_t numThreads);
    void sendNotification(const std::string &type, const std::string &message);

private:
    ThreadPool threadPool;
    NotificationHandler handler;
};

NotificationService::NotificationService(size_t numThreads) : threadPool(numThreads) {}

void NotificationService::sendNotification(const std::string &type, const std::string &message) {
    threadPool.enqueueTask([this, type, message] {
        if (type == "email") handler.sendEmail(message);
        else if (type == "sms") handler.sendSMS(message);
        else if (type == "push") handler.sendPush(message);
    });
}

// Main Driver Code
int main() {
    NotificationService notificationService(4);

    notificationService.sendNotification("email", "Hello, Email User!");
    notificationService.sendNotification("sms", "Hello, SMS User!");
    notificationService.sendNotification("push", "Hello, Push User!");

    std::this_thread::sleep_for(std::chrono::seconds(2)); // Allow threads to finish
    return 0;
}
```

## **🔹 Summary**  
✅ **High-Level & Low-Level Design** with **thread pool and queue**  
✅ **API Design** for real-world usage  
✅ **Modern C++ Implementation** using **multithreading, thread pooling, and lambda functions**  
✅ **Efficient & Scalable** Notification Service  

This C++ code efficiently handles notifications in **parallel** using a **thread pool**, ensuring high **performance** and **scalability**. 🚀
