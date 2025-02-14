# Logging and Monitoring System - C++ Implementation

## **Complete Code**

```cpp
#include <iostream>
#include <thread>
#include <vector>
#include <queue>
#include <functional>
#include <mutex>
#include <condition_variable>
#include <map>
#include <sqlite3.h>
#include <string>
#include <sstream>
#include <ctime>

// Utility to get current timestamp
std::string getCurrentTimestamp() {
    time_t now = time(nullptr);
    char buf[20];
    strftime(buf, sizeof(buf), "%Y-%m-%d %H:%M:%S", localtime(&now));
    return std::string(buf);
}

// Thread Pool Implementation
class ThreadPool {
public:
    ThreadPool(size_t threads) : stop(false) {
        for (size_t i = 0; i < threads; ++i) {
            workers.emplace_back([this] {
                while (true) {
                    std::function<void()> task;
                    {
                        std::unique_lock<std::mutex> lock(queueMutex);
                        condition.wait(lock, [this] { return stop || !tasks.empty(); });
                        if (stop && tasks.empty())
                            return;
                        task = std::move(tasks.front());
                        tasks.pop();
                    }
                    task();
                }
            });
        }
    }

    ~ThreadPool() {
        {
            std::unique_lock<std::mutex> lock(queueMutex);
            stop = true;
        }
        condition.notify_all();
        for (std::thread &worker : workers) {
            worker.join();
        }
    }

    template <class F>
    void enqueue(F&& f) {
        {
            std::unique_lock<std::mutex> lock(queueMutex);
            tasks.emplace(std::forward<F>(f));
        }
        condition.notify_one();
    }

private:
    std::vector<std::thread> workers;
    std::queue<std::function<void()>> tasks;
    std::mutex queueMutex;
    std::condition_variable condition;
    bool stop;
};

// Database Wrapper
class Database {
public:
    Database(const std::string& dbFile) {
        if (sqlite3_open(dbFile.c_str(), &db)) {
            throw std::runtime_error("Failed to open database");
        }
        setupTables();
    }

    ~Database() {
        sqlite3_close(db);
    }

    void executeQuery(const std::string& query) {
        char* errMsg = nullptr;
        if (sqlite3_exec(db, query.c_str(), nullptr, nullptr, &errMsg) != SQLITE_OK) {
            std::string error = errMsg;
            sqlite3_free(errMsg);
            throw std::runtime_error("SQL Error: " + error);
        }
    }

    void insertLog(const std::string& level, const std::string& message) {
        std::ostringstream query;
        query << "INSERT INTO logs (timestamp, level, message) VALUES ('"
              << getCurrentTimestamp() << "', '" << level << "', '" << message << "');";
        executeQuery(query.str());
    }

    void insertMetric(const std::string& metricName, double value) {
        std::ostringstream query;
        query << "INSERT INTO metrics (timestamp, metric_name, value) VALUES ('"
              << getCurrentTimestamp() << "', '" << metricName << "', " << value << ");";
        executeQuery(query.str());
    }

private:
    sqlite3* db;

    void setupTables() {
        executeQuery("CREATE TABLE IF NOT EXISTS logs ("
                     "id INTEGER PRIMARY KEY AUTOINCREMENT, "
                     "timestamp DATETIME DEFAULT CURRENT_TIMESTAMP, "
                     "level TEXT NOT NULL, "
                     "message TEXT NOT NULL);");
        executeQuery("CREATE TABLE IF NOT EXISTS metrics ("
                     "id INTEGER PRIMARY KEY AUTOINCREMENT, "
                     "timestamp DATETIME DEFAULT CURRENT_TIMESTAMP, "
                     "metric_name TEXT NOT NULL, "
                     "value REAL NOT NULL);");
    }
};

// Monitoring Engine
class MonitoringEngine {
public:
    MonitoringEngine(const std::map<std::string, double>& thresholds) : thresholds(thresholds) {}

    void checkMetric(const std::string& metricName, double value) {
        if (thresholds.count(metricName) && value > thresholds.at(metricName)) {
            std::cout << "[ALERT] " << metricName << " exceeded threshold with value: " << value << "
";
        }
    }

private:
    std::map<std::string, double> thresholds;
};

// Application Handlers
class Application {
public:
    Application(Database& db, MonitoringEngine& engine, ThreadPool& threadPool)
        : db(db), engine(engine), threadPool(threadPool) {}

    void handleLogRequest(const std::string& level, const std::string& message) {
        threadPool.enqueue([this, level, message] {
            db.insertLog(level, message);
            std::cout << "[INFO] Log stored: " << level << " - " << message << "
";
        });
    }

    void handleMetricRequest(const std::string& metricName, double value) {
        threadPool.enqueue([this, metricName, value] {
            db.insertMetric(metricName, value);
            engine.checkMetric(metricName, value);
            std::cout << "[INFO] Metric stored: " << metricName << " - " << value << "
";
        });
    }

private:
    Database& db;
    MonitoringEngine& engine;
    ThreadPool& threadPool;
};

// Main Function
int main() {
    try {
        Database db("logging_system.db");
        std::map<std::string, double> thresholds = {{"cpu_usage", 80.0}, {"memory_usage", 90.0}};
        MonitoringEngine engine(thresholds);
        ThreadPool threadPool(4);
        Application app(db, engine, threadPool);

        // Example Usage
        app.handleLogRequest("info", "Application started");
        app.handleLogRequest("error", "Disk space is low");
        app.handleMetricRequest("cpu_usage", 85.5);
        app.handleMetricRequest("memory_usage", 75.0);

        // Allow tasks to finish before exiting
        std::this_thread::sleep_for(std::chrono::seconds(1));
    } catch (const std::exception& ex) {
        std::cerr << "[ERROR] " << ex.what() << "
";
    }

    return 0;
}
```

## **Features**

1. **Multithreading**: Implements a thread pool to handle concurrent tasks efficiently.
2. **SQLite Database**: Manages log and metric storage.
3. **Monitoring Engine**: Monitors metrics against thresholds and generates alerts.
4. **Scalable and Modular**: Easily extendable with additional functionality.
