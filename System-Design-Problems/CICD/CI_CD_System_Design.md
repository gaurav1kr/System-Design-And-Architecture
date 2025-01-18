
# CI/CD System Design

This document outlines the low-level design (LLD), class diagram, and implementation of a CI/CD system using modern C++ features, including thread pools and templates.

---

## Low-Level Design (LLD)

### **Modules**
1. **Source Control Monitor**:
   - Detects changes in the source control (e.g., Git) repository.
   - Triggers pipeline on updates.

2. **Build System**:
   - Compiles the code.
   - Produces artifacts (binaries, libraries).

3. **Testing Framework**:
   - Runs unit tests, integration tests, etc.
   - Reports results.

4. **Artifact Storage**:
   - Stores built artifacts in a central location (e.g., S3, Nexus).

5. **Deployment Manager**:
   - Deploys the artifacts to target environments.

6. **Pipeline Orchestrator**:
   - Coordinates the flow between the modules.
   - Utilizes thread pools for parallel execution.

---

## Class Diagram

```plaintext
+-----------------------+
|   SourceControl       |
|-----------------------|
| + monitorRepo()       |
+-----------------------+
           |
           V
+-----------------------+
|    PipelineOrchestrator|
|-----------------------|
| + startPipeline()     |
| + addStage(Stage)     |
+-----------------------+
           |
           V
+-----------------------+      +----------------+
|        Stage          |<>----| ThreadPool     |
|-----------------------|      |----------------|
| + execute()           |      | + enqueueTask()|
|-----------------------|      +----------------+
           |
           V
+-----------------------+      +-------------------+
| BuildSystem           |      | TestingFramework |
|-----------------------|      |-------------------|
| + buildCode()         |      | + runTests()      |
+-----------------------+      +-------------------+
           |
           V
+-----------------------+      +-------------------+
| ArtifactStorage       |      | DeploymentManager|
|-----------------------|      |-------------------|
| + storeArtifacts()    |      | + deploy()        |
+-----------------------+      +-------------------+
```

---

## Implementation

### **Thread Pool Implementation**
```cpp
#include <iostream>
#include <vector>
#include <queue>
#include <thread>
#include <mutex>
#include <condition_variable>
#include <functional>

class ThreadPool {
    std::vector<std::thread> workers;
    std::queue<std::function<void()>> tasks;
    std::mutex queueMutex;
    std::condition_variable condition;
    bool stop;

public:
    ThreadPool(size_t threads) : stop(false) {
        for (size_t i = 0; i < threads; ++i) {
            workers.emplace_back([this]() {
                while (true) {
                    std::function<void()> task;
                    {
                        std::unique_lock<std::mutex> lock(this->queueMutex);
                        this->condition.wait(lock, [this]() { return this->stop || !this->tasks.empty(); });
                        if (this->stop && this->tasks.empty()) return;
                        task = std::move(this->tasks.front());
                        this->tasks.pop();
                    }
                    task();
                }
            });
        }
    }

    template <class F>
    void enqueueTask(F&& task) {
        {
            std::unique_lock<std::mutex> lock(queueMutex);
            tasks.emplace(std::forward<F>(task));
        }
        condition.notify_one();
    }

    ~ThreadPool() {
        {
            std::unique_lock<std::mutex> lock(queueMutex);
            stop = true;
        }
        condition.notify_all();
        for (std::thread& worker : workers) worker.join();
    }
};
```

### **Pipeline Components**

```cpp
#include <string>
#include <iostream>
#include <chrono>
#include <thread>

// Base Stage class
class Stage {
public:
    virtual void execute() = 0;
    virtual ~Stage() = default;
};

// Source Control Monitoring
class SourceControl : public Stage {
public:
    void execute() override {
        std::cout << "[SourceControl] Monitoring repository for changes...\n";
        std::this_thread::sleep_for(std::chrono::seconds(1));
    }
};

// Build System
class BuildSystem : public Stage {
public:
    void execute() override {
        std::cout << "[BuildSystem] Building code...\n";
        std::this_thread::sleep_for(std::chrono::seconds(2));
    }
};

// Testing Framework
class TestingFramework : public Stage {
public:
    void execute() override {
        std::cout << "[TestingFramework] Running tests...\n";
        std::this_thread::sleep_for(std::chrono::seconds(2));
    }
};

// Artifact Storage
class ArtifactStorage : public Stage {
public:
    void execute() override {
        std::cout << "[ArtifactStorage] Storing artifacts...\n";
        std::this_thread::sleep_for(std::chrono::seconds(1));
    }
};

// Deployment Manager
class DeploymentManager : public Stage {
public:
    void execute() override {
        std::cout << "[DeploymentManager] Deploying application...\n";
        std::this_thread::sleep_for(std::chrono::seconds(2));
    }
};
```

### **Pipeline Orchestrator**

```cpp
#include <vector>
#include <memory>

class PipelineOrchestrator {
    ThreadPool threadPool;
    std::vector<std::unique_ptr<Stage>> stages;

public:
    PipelineOrchestrator(size_t threads) : threadPool(threads) {}

    void addStage(std::unique_ptr<Stage> stage) {
        stages.emplace_back(std::move(stage));
    }

    void startPipeline() {
        for (auto& stage : stages) {
            threadPool.enqueueTask([stage = stage.get()]() {
                stage->execute();
            });
        }
    }
};

int main() {
    PipelineOrchestrator orchestrator(4);

    orchestrator.addStage(std::make_unique<SourceControl>());
    orchestrator.addStage(std::make_unique<BuildSystem>());
    orchestrator.addStage(std::make_unique<TestingFramework>());
    orchestrator.addStage(std::make_unique<ArtifactStorage>());
    orchestrator.addStage(std::make_unique<DeploymentManager>());

    orchestrator.startPipeline();

    // Allow time for pipeline to complete
    std::this_thread::sleep_for(std::chrono::seconds(10));
    return 0;
}
```

---

## Features in the Design

1. **Thread Pool**: Efficiently manages concurrent stages.
2. **Templates**: Used in the thread pool for task flexibility.
3. **Polymorphism**: Base `Stage` class for extensibility.
4. **Modern C++**: Uses smart pointers (`std::unique_ptr`) for resource management.
5. **Scalability**: Adding new stages or modifying existing ones is straightforward.
