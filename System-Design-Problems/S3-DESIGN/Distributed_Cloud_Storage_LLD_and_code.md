
# Distributed Cloud Storage System: LLD and Class Diagram

## Low-Level Design (LLD)

### Components and Interactions

1. **ThreadPool**
   - **Responsibilities**: Manages a pool of worker threads to execute tasks asynchronously.
   - **Key Interactions**:
     - Executes upload and download tasks from `S3Server`.

2. **StorageNode**
   - **Responsibilities**: Stores and retrieves file data.
   - **Key Interactions**:
     - Receives file data from `S3Server` for storage.
     - Serves file content to `S3Server` on retrieval requests.
     - Interacts with `ReplicationManager` for replication.

3. **MetadataManager**
   - **Responsibilities**: Tracks file locations and metadata.
   - **Key Interactions**:
     - Updates metadata during file uploads via `S3Server`.
     - Retrieves file location during downloads via `S3Server`.

4. **ReplicationManager**
   - **Responsibilities**: Manages file replication across multiple storage nodes. Periodically checks health of nodes.
   - **Key Interactions**:
     - Coordinates with `StorageNode` to replicate files.
     - Uses `S3Server` to monitor storage node health.

5. **S3Server**
   - **Responsibilities**: Acts as the central server, managing file uploads, downloads, and replication.
   - **Key Interactions**:
     - Enqueues tasks to `ThreadPool`.
     - Uploads files to primary `StorageNode` and triggers replication via `ReplicationManager`.
     - Retrieves file metadata from `MetadataManager` during downloads.

---

## Class Diagram

```
+------------------+          +------------------+
|   ThreadPool     |          |   StorageNode    |
+------------------+          +------------------+
| - workers        |          | - data           |
| - tasks          |          | - dataMutex      |
| - queueMutex     |          | - isHealthy      |
| - condition      |          +------------------+
+------------------+          | + storeFile()    |
| + enqueue()      |          | + retrieveFile() |
+------------------+          | + checkHealth()  |
                              +------------------+
       ^                              ^
       |                              |
       |                              |
+------------------+          +-----------------------+
|    S3Server      |<-------->|  ReplicationManager   |
+------------------+          +-----------------------+
| - threadPool     |          | - storageNodes        |
| - storageNodes   |          | - replicationFactor   |
| - metadataMgr    |          | - replicationMutex    |
| - replicationMgr |          +-----------------------+
+------------------+          | + replicateFile()     |
| + uploadFile()   |          | + checkNodeHealth()   |
| + downloadFile() |          +-----------------------+
+------------------+
       |
       |
       v
+------------------+
|  MetadataManager |
+------------------+
| - metadata       |
| - metadataMutex  |
+------------------+
| + addFile()      |
| + getFileLocation()|
+------------------+
```

---

## Relationships Between Classes

1. **Composition**:
   - `S3Server` *composes* `ThreadPool`, `MetadataManager`, and `ReplicationManager`.
   - `ReplicationManager` *composes* a collection of `StorageNode` objects.

2. **Aggregation**:
   - `ReplicationManager` has a many-to-many relationship with `StorageNode`.

3. **Direct Interaction**:
   - `S3Server` interacts directly with `ThreadPool` for task management.
   - `S3Server` interacts with `MetadataManager` for file metadata and location retrieval.

4. **Flow**:
   - File upload/download operations originate in `S3Server` and cascade down to `StorageNode` and `ReplicationManager` via `ThreadPool`.

---

## Explanation

1. **ThreadPool** ensures tasks (upload/download) are non-blocking and asynchronous.
2. **StorageNode** provides physical storage with health monitoring.
3. **MetadataManager** tracks which node stores which file.
4. **ReplicationManager** ensures fault tolerance by replicating data across multiple nodes and maintaining node health checks.
5. **S3Server** coordinates everything, acting as the main entry point.

---
### C++ code
```
//main.cpp
#include "S3Server.h"
#include <iostream>
#include <thread>
#include <chrono>

int main() 
{
    S3Server server(4, 3, 2);

    server.uploadFile("file1.txt", "Hello, Cloud!");
    server.uploadFile("file2.txt", "Replication Works!");
    server.downloadFile("file1.txt");
    server.downloadFile("file2.txt");

    std::this_thread::sleep_for(std::chrono::seconds(2)); // Allow async tasks to complete
    return 0;
}
```

```
//SEServer.h
#ifndef S3SERVER_H
#define S3SERVER_H

#include "ThreadPool.h"
#include "StorageNode.h"
#include "MetadataManager.h"
#include "ReplicationManager.h"

class S3Server 
{
private:
    ThreadPool threadPool;
    MetadataManager metadataManager;
    std::vector<std::shared_ptr<StorageNode>> storageNodes;
    ReplicationManager replicationManager;

public:
    S3Server(size_t threadCount, size_t nodeCount, size_t replicationFactor)
        : threadPool(threadCount),replicationManager(storageNodes, replicationFactor) 
    {
        for (size_t i = 0; i < nodeCount; ++i) 
        {
            storageNodes.push_back(std::make_shared<StorageNode>());
        }
    }

    void uploadFile(const std::string& filename, const std::string& content) 
    {
        threadPool.enqueue([this, filename, content]() 
            {
            size_t primaryNodeIndex = std::hash<std::string>{}(filename) % storageNodes.size();
            storageNodes[primaryNodeIndex]->storeFile(filename, content);
            metadataManager.addFile(filename, "Node_" + std::to_string(primaryNodeIndex));
            replicationManager.replicateFile(filename, content);
            });
    }

    void downloadFile(const std::string& filename) 
    {
        threadPool.enqueue([this, filename]() 
            {
            try {
                std::string location = metadataManager.getFileLocation(filename);
                size_t nodeIndex = std::stoi(location.substr(5));
                std::string content = storageNodes[nodeIndex]->retrieveFile(filename);
                std::cout << "File downloaded: " << filename << " Content: " << content << "\n";
            }
            catch (const std::exception& e) {
                std::cerr << "Error: " << e.what() << "\n";
            }
            });
    }
};

#endif // S3SERVER_H
```

```
//ReplicationManager.h
#ifndef REPLICATIONMANAGER_H
#define REPLICATIONMANAGER_H

#include "StorageNode.h"
#include <vector>
#include <string>
#include <mutex>
#include <iostream>

class ReplicationManager 
{
private:
    std::vector<std::shared_ptr<StorageNode>> storageNodes;
    size_t replicationFactor;
    std::mutex replicationMutex;

public:
    ReplicationManager(std::vector<std::shared_ptr<StorageNode>> nodes, size_t factor)
        : storageNodes(std::move(nodes)), replicationFactor(factor) {}

    void replicateFile(const std::string& filename, const std::string& content) {
        std::lock_guard<std::mutex> lock(replicationMutex);
        size_t nodeCount = storageNodes.size();
        for (size_t i = 0; i < replicationFactor; ++i) {
            size_t nodeIndex = (std::hash<std::string>{}(filename)+i) % nodeCount;
            storageNodes[nodeIndex]->storeFile(filename, content);
        }
    }
};

#endif // REPLICATIONMANAGER_H
```

```
//MetadataManager.h
#ifndef METADATAMANAGER_H
#define METADATAMANAGER_H

#include <unordered_map>
#include <string>
#include <mutex>

class MetadataManager 
{
private:
    std::unordered_map<std::string, std::string> metadata; // filename -> node location
    std::mutex metadataMutex;

public:
    void addFile(const std::string& filename, const std::string& location) 
	{
        std::lock_guard<std::mutex> lock(metadataMutex);
        metadata[filename] = location;
    }

    std::string getFileLocation(const std::string& filename) 
	{
        std::lock_guard<std::mutex> lock(metadataMutex);
        if (metadata.find(filename) != metadata.end()) {
            return metadata[filename];
        }
        throw std::runtime_error("File not found in metadata");
    }
};

#endif // METADATAMANAGER_H
```

```
//StorageNode.h
#ifndef STORAGENODE_H
#define STORAGENODE_H

#include <unordered_map>
#include <string>
#include <mutex>
#include <atomic>
#include <stdexcept>
#include <iostream>

class StorageNode 
{
private:
    std::unordered_map<std::string, std::string> data;
    std::mutex dataMutex;
    std::atomic<bool> isHealthy;

public:
    StorageNode() : isHealthy(true) {}

    void storeFile(const std::string& filename, const std::string& content) 
	{
        std::lock_guard<std::mutex> lock(dataMutex);
        data[filename] = content;
    }

    std::string retrieveFile(const std::string& filename) 
	{
        std::lock_guard<std::mutex> lock(dataMutex);
        if (data.find(filename) != data.end()) {
            return data[filename];
        }
        throw std::runtime_error("File not found");
    }

    void markUnhealthy() { isHealthy.store(false); }
    void markHealthy() { isHealthy.store(true); }
    bool checkHealth() const { return isHealthy.load(); }
};

#endif // STORAGENODE_H
```

```
//ThreadPool.h
#ifndef THREADPOOL_H
#define THREADPOOL_H

#include <vector>
#include <queue>
#include <thread>
#include <mutex>
#include <condition_variable>
#include <functional>
#include <atomic>

class ThreadPool {
private:
    std::vector<std::thread> workers;
    std::queue<std::function<void()>> tasks;
    std::mutex queueMutex;
    std::condition_variable condition;
    std::atomic<bool> stop;

public:
    explicit ThreadPool(size_t threadCount) : stop(false) 
    {
        for (size_t i = 0; i < threadCount; ++i) 
        {
            workers.emplace_back([this]() 
                {
                while (true) 
                {
                    std::function<void()> task;
                    {
                        std::unique_lock<std::mutex> lock(queueMutex);
                        condition.wait(lock, [this]() { return stop || !tasks.empty(); });
                        if (stop && tasks.empty()) return;
                        task = std::move(tasks.front());
                        tasks.pop();
                    }
                    task();
                }
                });
        }
    }

    ~ThreadPool() 
    {
        stop.store(true);
        condition.notify_all();
        for (auto& worker : workers) 
        {
            if (worker.joinable()) worker.join();
        }
    }

    void enqueue(std::function<void()> task) 
    {
        {
            std::lock_guard<std::mutex> lock(queueMutex);
            tasks.push(std::move(task));
        }
        condition.notify_one();
    }
};

#endif // THREADPOOL_H

```