
# Distributed Locking Service Design

## 1. High-Level Design (HLD)
**Objective:** Design a service that allows multiple distributed clients to acquire and release locks on shared resources safely.

### Components:
1. **Lock Service**: Maintains lock states and synchronizes access across clients. It uses a distributed database like etcd or Redis to store lock states.
2. **Lock Manager**: Manages lock requests from clients. Ensures only one client holds the lock on a resource.
3. **Client Library**: Used by distributed clients to interact with the Lock Service.

### Workflow:
1. Client requests a lock from the Lock Service via the Lock Manager.
2. Lock Manager checks the lock status in the database and, if free, grants it.
3. Client holds the lock and performs operations, then releases the lock.
4. Lock Manager updates the lock state to make it available for other clients.

## 2. Low-Level Design (LLD)
### Data Flow:
- Each lock is identified by a unique key.
- Lock state is tracked in the database (locked/unlocked).
- A timeout mechanism releases stale locks if the client crashes or fails to release.

### Fault Tolerance:
- Use distributed consensus (e.g., etcd’s Raft) to ensure lock status consistency.
- Heartbeat mechanism to maintain lock validity.

## 3. Schema Design
The schema for a basic distributed lock table might include:
- `lock_id` (String, Primary Key): Unique identifier for each lock.
- `owner_id` (String): ID of the client currently holding the lock.
- `timestamp` (DateTime): The time when the lock was acquired.
- `expiration_time` (DateTime): Time at which the lock expires.
- `status` (Boolean): Current status (locked/unlocked).

## 4. UML Diagram
The UML diagram shows relationships between:
1. **Client**: Requests and releases locks.
2. **LockManager**: Processes lock/unlock requests.
3. **LockService**: Core service providing distributed lock functionality.
4. **Database**: Stores lock information and ensures state consistency.

## 5. Class Diagram
### Classes:
1. **LockManager**: Handles lock requests.
2. **DistributedLock**: Represents a lock with methods for acquire/release.
3. **LockDatabase**: Interface for database operations.
4. **Client**: Interacts with LockManager.

## 6. Sequence Diagram
1. **Lock Acquisition**:
   - Client requests lock → LockManager verifies status → LockDatabase updates status → Lock granted.
2. **Lock Release**:
   - Client releases lock → LockManager updates status in LockDatabase → Lock available.

## 7. C++ Code Implementation

Here’s a simplified C++ implementation:

```cpp
#include <iostream>
#include <map>
#include <mutex>
#include <string>
#include <chrono>
#include <thread>

class LockDatabase {
public:
    bool acquireLock(const std::string& lockId, const std::string& ownerId) {
        std::unique_lock<std::mutex> lock(dbMutex);
        auto it = locks.find(lockId);
        if (it == locks.end() || !it->second.locked) {
            locks[lockId] = {ownerId, true, getCurrentTime()};
            return true;
        }
        return false;
    }

    bool releaseLock(const std::string& lockId, const std::string& ownerId) {
        std::unique_lock<std::mutex> lock(dbMutex);
        auto it = locks.find(lockId);
        if (it != locks.end() && it->second.ownerId == ownerId) {
            it->second.locked = false;
            return true;
        }
        return false;
    }

private:
    struct LockInfo {
        std::string ownerId;
        bool locked;
        std::chrono::time_point<std::chrono::steady_clock> timestamp;
    };

    std::map<std::string, LockInfo> locks;
    std::mutex dbMutex;

    std::chrono::time_point<std::chrono::steady_clock> getCurrentTime() {
        return std::chrono::steady_clock::now();
    }
};

class LockManager {
public:
    LockManager(LockDatabase& db) : database(db) {}

    bool acquire(const std::string& lockId, const std::string& clientId) {
        return database.acquireLock(lockId, clientId);
    }

    bool release(const std::string& lockId, const std::string& clientId) {
        return database.releaseLock(lockId, clientId);
    }

private:
    LockDatabase& database;
};

class Client {
public:
    Client(LockManager& manager, const std::string& id) : lockManager(manager), clientId(id) {}

    void doWorkWithLock(const std::string& lockId) {
        if (lockManager.acquire(lockId, clientId)) {
            std::cout << "Client " << clientId << " acquired lock on " << lockId << "\n";
            std::this_thread::sleep_for(std::chrono::seconds(2)); // Simulate work
            lockManager.release(lockId, clientId);
            std::cout << "Client " << clientId << " released lock on " << lockId << "\n";
        } else {
            std::cout << "Client " << clientId << " could not acquire lock on " << lockId << "\n";
        }
    }

private:
    LockManager& lockManager;
    std::string clientId;
};

int main() {
    LockDatabase db;
    LockManager manager(db);

    Client client1(manager, "Client1");
    Client client2(manager, "Client2");

    std::thread t1(&Client::doWorkWithLock, &client1, "resource1");
    std::thread t2(&Client::doWorkWithLock, &client2, "resource1");

    t1.join();
    t2.join();

    return 0;
}
```

---

This markdown file contains all the requested information for the Distributed Locking Service Design.
