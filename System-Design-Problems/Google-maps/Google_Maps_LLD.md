
# Google Maps Low-Level Design

This document describes the low-level design (LLD) of a simplified Google Maps system, including its core classes, interaction model, and C++ implementation.

---

## **Textual Class Diagram**
### Core Classes:
1. **Map**
   - **Attributes**: 
     - `Graph<Place>`
     - `std::unordered_map<std::string, Place>`
   - **Methods**: 
     - `addPlace`
     - `addRoad`
     - `findShortestPath`
     - `searchPlace`

2. **Place**
   - **Attributes**: 
     - `std::string name`
     - `double latitude`
     - `double longitude`
   - **Methods**: None (data class)

3. **Graph<T>**
   - **Attributes**: 
     - `std::unordered_map<T, std::vector<std::pair<T, double>>>`
   - **Methods**: 
     - `addEdge`
     - `getNeighbors`

4. **PathFinder**
   - **Methods**: 
     - `dijkstra`

5. **ThreadPool**
   - **Attributes**: 
     - `std::vector<std::thread>`
     - `std::queue<std::function<void()>>`
     - `std::mutex`
     - `std::condition_variable`
     - `bool stop`
   - **Methods**: 
     - `enqueue`
     - `start`
     - `stop`

---

## Interaction Among Classes:
1. **Map** interacts with **Place** to maintain information about locations and their relationships.
2. **Map** uses **Graph** to store the road network.
3. **Map** uses **PathFinder** to calculate optimal routes between places.
4. **ThreadPool** is used to parallelize tasks, such as searching for places or finding multiple routes.

---

## **Code Implementation**
Below is the detailed C++ implementation:

```cpp
#include <iostream>
#include <unordered_map>
#include <vector>
#include <string>
#include <queue>
#include <mutex>
#include <condition_variable>
#include <thread>
#include <functional>
#include <cmath>
#include <limits>
#include <optional>

// Place class
class Place 
{
public:
    std::string name;
    double latitude;
    double longitude;

    Place(const std::string& name, double lat, double lon) 
        : name(name), latitude(lat), longitude(lon) {}
};

// Graph class
template <typename T>
class Graph 
{
    std::unordered_map<T, std::vector<std::pair<T, double>>> adjList;

public:
    void addEdge(const T& src, const T& dest, double weight) 
	{
        adjList[src].emplace_back(dest, weight);
        adjList[dest].emplace_back(src, weight); // Undirected graph
    }

    const std::vector<std::pair<T, double>>& getNeighbors(const T& node) const 
	{
        return adjList.at(node);
    }
};

// PathFinder class
class PathFinder 
{
public:
    template <typename T>
    std::vector<T> dijkstra(const Graph<T>& graph, const T& start, const T& goal) 
	{
        std::unordered_map<T, double> distances;
        std::unordered_map<T, T> previous;
        std::priority_queue<std::pair<double, T>, std::vector<std::pair<double, T>>, std::greater<>> pq;

        distances[start] = 0.0;
        pq.push({0.0, start});

        while (!pq.empty()) 
		{
            auto [dist, current] = pq.top();
            pq.pop();

            if (current == goal) break;

            for (const auto& [neighbor, weight] : graph.getNeighbors(current)) 
			{
                double newDist = dist + weight;
                if (newDist < distances[neighbor]) {
                    distances[neighbor] = newDist;
                    previous[neighbor] = current;
                    pq.push({newDist, neighbor});
                }
            }
        }

        // Backtrack to find the path
        std::vector<T> path;
        for (T at = goal; previous.find(at) != previous.end(); at = previous[at]) 
		{
            path.push_back(at);
        }
        path.push_back(start);
        std::reverse(path.begin(), path.end());
        return path;
    }
};

// ThreadPool class
class ThreadPool 
{
    std::vector<std::thread> workers;
    std::queue<std::function<void()>> tasks;
    std::mutex queueMutex;
    std::condition_variable condition;
    bool stop = false;

public:
    ThreadPool(size_t threads) 
	{
        for (size_t i = 0; i < threads; ++i) 
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

    template <class F>
    void enqueue(F&& task) 
	{
        {
            std::unique_lock<std::mutex> lock(queueMutex);
            tasks.emplace(std::forward<F>(task));
        }
        condition.notify_one();
    }

    ~ThreadPool() 
	{
        {
            std::unique_lock<std::mutex> lock(queueMutex);
            stop = true;
        }
        condition.notify_all();
        for (std::thread& worker : workers) {
            worker.join();
        }
    }
};

// Map class
class Map 
{
    Graph<std::string> roadNetwork;
    std::unordered_map<std::string, Place> places;

public:
    void addPlace(const std::string& name, double latitude, double longitude) 
	{
        places[name] = Place(name, latitude, longitude);
    }

    void addRoad(const std::string& from, const std::string& to, double distance) 
	{
        roadNetwork.addEdge(from, to, distance);
    }

    std::vector<std::string> findShortestPath(const std::string& start, const std::string& goal) 
	{
        PathFinder pathFinder;
        return pathFinder.dijkstra(roadNetwork, start, goal);
    }
};

// Main Function
int main() 
{
    Map map;
    map.addPlace("A", 12.9716, 77.5946);
    map.addPlace("B", 13.0827, 80.2707);
    map.addPlace("C", 11.0168, 76.9558);

    map.addRoad("A", "B", 5.0);
    map.addRoad("A", "C", 10.0);
    map.addRoad("B", "C", 3.0);

    ThreadPool threadPool(4);
    threadPool.enqueue([&]() {
        auto path = map.findShortestPath("A", "C");
        std::cout << "Shortest path: ";
        for (const auto& place : path) {
            std::cout << place << " ";
        }
        std::cout << std::endl;
    });

    return 0;
}
```

---

## Key Features:
1. **Modern C++ Concepts**:
   - Use of `std::function`, `std::thread`, `std::mutex`, and `std::condition_variable`.
   - STL containers like `std::unordered_map`, `std::vector`, `std::priority_queue`.

2. **Thread Pool**:
   - Parallelizes tasks such as finding multiple routes or processing large data.

3. **Modular Design**:
   - Clear separation of responsibilities between classes.
   - Scalable design to add new features like real-time traffic updates.

4. **Efficient Pathfinding**:
   - Uses Dijkstra's algorithm for finding shortest paths.

---

Let me know if you need any further clarification or additional features!
