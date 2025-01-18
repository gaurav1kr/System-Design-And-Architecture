
# Stability Monitor for Kubernetes Pods

## **Overview**

The Stability Monitor is designed to monitor Kubernetes pods' resource usage (CPU, memory, flash) in a cluster, raise alarms when resources exceed thresholds, and track pod states (e.g., crash or inability to restart). This design leverages **modern C++ concepts**, including multithreading and STL, to handle concurrent monitoring of multiple pods.

---

## **LLD Overview**

### **Core Components**
1. **StabilityMonitor**: Central class managing monitoring tasks.
2. **ResourceMonitor**: Abstract base class for monitoring resources.
3. **CPUMonitor**, **MemoryMonitor**, **FlashMonitor**: Derived classes for monitoring CPU, memory, and flash usage, respectively.
4. **PodStateMonitor**: Class to monitor pod states and detect crashes or restarts.
5. **AlarmManager**: Class for handling alarm generation.
6. **Pod**: Class representing a Kubernetes pod with relevant metadata.

### **Multithreading**
- Thread pool for concurrently monitoring resources and pod states.

### **Alarms**
- Alarm thresholds for CPU, memory, flash usage, and pod crashes.
- Logging system for warnings and critical alarms.

### **External Interfaces**
- Use Kubernetes API to fetch pod metrics and states.

---

## **Class Diagram**

```plaintext
+----------------+         +----------------+         +------------------+
| StabilityMonitor|<>------| ResourceMonitor|<|-------| CPUMonitor       |
|                |         |                |         +------------------+
| - monitors     |         | + monitor()    |<|-------| MemoryMonitor    |
| - alarmManager |         +----------------+         +------------------+
| - threadPool   |         | + setThreshold()         | FlashMonitor     |
| + start()      |         +----------------+         +------------------+
| + stop()       |                                     +------------------+
| + logAlarm()   |
+----------------+
      |                                  +------------------+
      +--------------------------------->| AlarmManager     |
                                         | + generateAlarm()|
                                         +------------------+

+----------------+                      +------------------+
| Pod            |<>--------------------| PodStateMonitor  |
|                |                      | + monitorStates()|
| - podName      |                      +------------------+
| - podID        |
| - cpuUsage     |
| - memoryUsage  |
| - flashUsage   |
| - state        |
+----------------+
```

---

## **Low-Level Design Details**

### **1. StabilityMonitor**
- Manages all monitoring tasks and coordinates multithreading for concurrent monitoring.
- Maintains a list of `Pod` objects representing the cluster.

```cpp
class StabilityMonitor {
private:
    std::vector<std::shared_ptr<Pod>> pods;
    std::unique_ptr<AlarmManager> alarmManager;
    std::vector<std::thread> threadPool;
    std::atomic<bool> running;

    void monitorResource(std::shared_ptr<ResourceMonitor> monitor);
    void monitorPodStates();

public:
    StabilityMonitor();
    ~StabilityMonitor();

    void start();
    void stop();
};
```

---

### **2. ResourceMonitor (Abstract Base Class)**
- Base class for resource monitoring (CPU, memory, flash).

```cpp
class ResourceMonitor {
protected:
    double threshold;
    StabilityMonitor& parentMonitor;

    virtual void checkUsage(const Pod& pod) = 0;

public:
    ResourceMonitor(StabilityMonitor& monitor, double threshold);
    virtual ~ResourceMonitor() {}

    void monitor(const std::vector<std::shared_ptr<Pod>>& pods);
};
```

---

### **3. CPUMonitor, MemoryMonitor, FlashMonitor**
- Derived classes implementing resource-specific checks.

```cpp
class CPUMonitor : public ResourceMonitor {
protected:
    void checkUsage(const Pod& pod) override;

public:
    CPUMonitor(StabilityMonitor& monitor, double threshold);
};

class MemoryMonitor : public ResourceMonitor {
protected:
    void checkUsage(const Pod& pod) override;

public:
    MemoryMonitor(StabilityMonitor& monitor, double threshold);
};

class FlashMonitor : public ResourceMonitor {
protected:
    void checkUsage(const Pod& pod) override;

public:
    FlashMonitor(StabilityMonitor& monitor, double threshold);
};
```

---

### **4. PodStateMonitor**
- Monitors pod states for crashes or inability to restart.

```cpp
class PodStateMonitor {
private:
    StabilityMonitor& parentMonitor;

public:
    PodStateMonitor(StabilityMonitor& monitor);

    void monitorStates(const std::vector<std::shared_ptr<Pod>>& pods);
};
```

---

### **5. Pod**
- Represents a Kubernetes pod and its metrics.

```cpp
class Pod {
public:
    std::string podName;
    std::string podID;
    double cpuUsage;
    double memoryUsage;
    double flashUsage;
    std::string state; // Running, CrashLoopBackOff, etc.

    Pod(const std::string& name, const std::string& id);
};
```

---

### **6. AlarmManager**
- Handles alarm generation and logging.

```cpp
class AlarmManager {
public:
    void generateAlarm(const std::string& message);
};
```

---

## **C++ Implementation**

### **StabilityMonitor Implementation**
```cpp
#include <iostream>
#include <thread>
#include <vector>
#include <atomic>
#include <memory>
#include "Pod.h"
#include "ResourceMonitor.h"
#include "AlarmManager.h"
#include "PodStateMonitor.h"

StabilityMonitor::StabilityMonitor()
    : running(false), alarmManager(std::make_unique<AlarmManager>()) {}

StabilityMonitor::~StabilityMonitor() {
    stop();
}

void StabilityMonitor::start() {
    running = true;

    // Start resource monitors
    auto cpuMonitor = std::make_shared<CPUMonitor>(*this, 80.0);
    auto memoryMonitor = std::make_shared<MemoryMonitor>(*this, 75.0);
    auto flashMonitor = std::make_shared<FlashMonitor>(*this, 90.0);

    threadPool.emplace_back(&StabilityMonitor::monitorResource, this, cpuMonitor);
    threadPool.emplace_back(&StabilityMonitor::monitorResource, this, memoryMonitor);
    threadPool.emplace_back(&StabilityMonitor::monitorResource, this, flashMonitor);

    // Start pod state monitoring
    threadPool.emplace_back(&StabilityMonitor::monitorPodStates, this);
}

void StabilityMonitor::stop() {
    running = false;
    for (auto& thread : threadPool) {
        if (thread.joinable()) {
            thread.join();
        }
    }
}

void StabilityMonitor::monitorResource(std::shared_ptr<ResourceMonitor> monitor) {
    while (running) {
        monitor->monitor(pods);
        std::this_thread::sleep_for(std::chrono::seconds(5));
    }
}

void StabilityMonitor::monitorPodStates() {
    PodStateMonitor podStateMonitor(*this);
    while (running) {
        podStateMonitor.monitorStates(pods);
        std::this_thread::sleep_for(std::chrono::seconds(10));
    }
}
```

---

### **CPUMonitor Implementation**
```cpp
void CPUMonitor::checkUsage(const Pod& pod) {
    if (pod.cpuUsage > threshold) {
        parentMonitor.logAlarm("CPU usage exceeded for Pod: " + pod.podName);
    }
}

CPUMonitor::CPUMonitor(StabilityMonitor& monitor, double threshold)
    : ResourceMonitor(monitor, threshold) {}
```

---

This design ensures **scalability** (using multithreading), **extensibility** (easily add new monitors), and **fault detection**.
