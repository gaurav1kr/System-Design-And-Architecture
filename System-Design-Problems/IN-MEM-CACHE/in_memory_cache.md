# In-Memory Cache System Design

## **High-Level Design (HLD)**

### **Requirements**
1. **Functional Requirements**
   - Store and retrieve key-value pairs.
   - Set expiration time for cache entries.
   - Support concurrency with multiple readers/writers.
   - Eviction policies: **Least Recently Used (LRU), Least Frequently Used (LFU)**.
   - Thread-safe operations.

2. **Non-Functional Requirements**
   - **High performance**: Low latency for reads and writes.
   - **Scalability**: Should handle multiple threads efficiently.
   - **Concurrency**: Use thread-safe data structures.
   - **Memory-efficient**: Avoid excessive memory usage.

### **Architecture**
- **Client API**: Interface for applications to interact with the cache.
- **Cache Manager**: Manages storage, retrieval, and eviction.
- **Eviction Policy**: Handles cache eviction strategies (LRU, LFU).
- **Background Threads**: 
  - **Thread Pool**: Manages worker threads efficiently.
  - **Expiration Thread**: Periodically removes expired cache items.

---

## **Low-Level Design (LLD)**

### **Classes & Components**
1. **CacheEntry**: Represents an individual cache item with key, value, and expiration time.
2. **CacheManager**: Manages insertion, retrieval, and eviction.
3. **EvictionPolicy**: Implements LRU eviction strategy.
4. **ThreadPool**: Manages worker threads for concurrent access.
5. **Cache API**: Provides an interface to clients.

---

## **API Design**

### **Cache API Endpoints**
| API | Description |
|------|------------|
| `put(key, value, ttl)` | Stores a key-value pair with optional expiration time (TTL). |
| `get(key)` | Retrieves the value of a key if it exists. |
| `remove(key)` | Deletes a key from the cache. |
| `size()` | Returns the number of items in the cache. |

---

## **C++ Implementation**

```cpp
#include <iostream>
#include <unordered_map>
#include <list>
#include <mutex>
#include <thread>
#include <condition_variable>
#include <chrono>
#include <vector>
#include <functional>

// Cache Entry
struct CacheEntry {
    std::string key;
    std::string value;
    std::chrono::steady_clock::time_point expiry;
};

// Thread-Safe LRU Cache
class LRUCache {
private:
    size_t capacity;
    std::list<CacheEntry> cacheList;
    std::unordered_map<std::string, std::list<CacheEntry>::iterator> cacheMap;
    std::mutex cacheMutex;

public:
    LRUCache(size_t cap) : capacity(cap) {}

    void put(const std::string& key, const std::string& value, int ttl = 0) {
        std::lock_guard<std::mutex> lock(cacheMutex);

        if (cacheMap.find(key) != cacheMap.end()) {
            cacheList.erase(cacheMap[key]);
        } else if (cacheList.size() >= capacity) {
            cacheMap.erase(cacheList.back().key);
            cacheList.pop_back();
        }

        CacheEntry entry{key, value, ttl > 0 ? std::chrono::steady_clock::now() + std::chrono::seconds(ttl) : std::chrono::steady_clock::time_point()};
        cacheList.push_front(entry);
        cacheMap[key] = cacheList.begin();
    }

    std::string get(const std::string& key) {
        std::lock_guard<std::mutex> lock(cacheMutex);

        if (cacheMap.find(key) == cacheMap.end())
            return "Not Found";

        auto it = cacheMap[key];

        if (it->expiry != std::chrono::steady_clock::time_point() && std::chrono::steady_clock::now() > it->expiry) {
            cacheList.erase(it);
            cacheMap.erase(key);
            return "Expired";
        }

        cacheList.splice(cacheList.begin(), cacheList, it);
        return it->value;
    }
};
```

---

## **Further Enhancements**
1. **Add Distributed Caching**: Use Redis or Memcached as a backend.
2. **Improve Eviction Policy**: Implement **LFU (Least Frequently Used)**.
3. **Support Persistent Storage**: Store data in disk-based DB.

This implementation provides **scalable, thread-safe, and high-performance in-memory caching** with **modern C++ techniques**.
