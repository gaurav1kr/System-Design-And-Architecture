# Terminal/Command Prompt Interface Design

This document provides the design specifications and implementation details for a Terminal/Command Prompt Interface. The design leverages multithreading, thread pooling, and modern C++ concepts.

---

## High-Level Design (HLD)

### Purpose
A command-line terminal interface that supports user input for executing commands in a multithreaded environment. Each command will execute as a task in a thread pool.

### Core Components
- **Command Parser**: Parses user input and identifies commands.
- **Thread Pool**: Manages and executes commands concurrently.
- **Command Executor**: Handles the execution of supported commands.
- **Result Manager**: Outputs results or logs errors.

### Flow
1. **User Input**: The user enters a command.
2. **Parsing**: The Command Parser validates and parses the input.
3. **Task Enqueue**: The parsed command is pushed to the Thread Pool.
4. **Execution**: The Thread Pool assigns the task to an available thread.
5. **Result Display**: The Command Executor processes the command and the Result Manager displays the output.
6. **Repeat/Exit**: The process continues until the user inputs an exit command.

---

## Low-Level Design (LLD)

### Classes and Their Responsibilities

- **`Terminal`**
  - Acts as the main interface for user input and output.
  - Coordinates parsing and task delegation to the thread pool.
  
- **`CommandParser`**
  - Parses user input into a command and its associated arguments.
  
- **`ThreadPool`**
  - Manages a fixed number of worker threads.
  - Uses a thread-safe queue to manage tasks.
  - Utilizes `std::thread`, `std::mutex`, and `std::condition_variable` for synchronization.
  
- **`CommandExecutor`**
  - Executes the parsed commands.
  - Supports commands such as `echo` and `help`.

- **`Task`**
  - Encapsulates a command to be executed using `std::function`.

### Multithreading Approach
- Utilize `std::thread` for creating worker threads.
- Use `std::condition_variable` to avoid busy waiting.
- The thread pool manages a shared task queue and distributes tasks among available threads.

### Modern C++ Concepts
- **Smart Pointers**: Use `std::unique_ptr` and `std::shared_ptr` for resource management.
- **Lambda Functions & `std::function`**: Allow flexible task definition and encapsulation.
- **Atomic Variables**: Use `std::atomic` to manage thread-safe flags.

---

## APIs

### `Terminal`
- **`void start()`**: Starts the terminal interface and processes user commands.

### `CommandParser`
- **`std::pair<std::string, std::vector<std::string>> parse(const std::string &input)`**: Parses the input string into a command and its arguments.

### `ThreadPool`
- **Constructor**: `ThreadPool(size_t numThreads)` initializes the thread pool with a specified number of threads.
- **`void enqueue(std::function<void()> task)`**: Adds a task to the queue for execution by worker threads.

### `CommandExecutor`
- **`void execute(const std::string &command, const std::vector<std::string> &args)`**: Executes a command based on the input and arguments.

### `Task`
- **`std::function<void()> getTask()`**: Returns the encapsulated task (not explicitly implemented as a separate class in this design, but the concept is embedded in using `std::function<void()>` for tasks).

---

## Working C++ Code

```cpp
#include <iostream>
#include <sstream>
#include <string>
#include <vector>
#include <queue>
#include <thread>
#include <mutex>
#include <condition_variable>
#include <functional>
#include <atomic>

// ThreadPool class definition
class ThreadPool {
private:
    std::vector<std::thread> workers;
    std::queue<std::function<void()>> tasks;
    std::mutex queueMutex;
    std::condition_variable condition;
    std::atomic<bool> stop;

public:
    explicit ThreadPool(size_t numThreads) : stop(false) {
        for (size_t i = 0; i < numThreads; ++i) {
            workers.emplace_back([this]() {
                while (true) {
                    std::function<void()> task;
                    {
                        std::unique_lock<std::mutex> lock(queueMutex);
                        condition.wait(lock, [this]() { return stop || !tasks.empty(); });

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
        stop = true;
        condition.notify_all();
        for (std::thread &worker : workers) {
            worker.join();
        }
    }

    void enqueue(std::function<void()> task) {
        {
            std::lock_guard<std::mutex> lock(queueMutex);
            tasks.emplace(std::move(task));
        }
        condition.notify_one();
    }
};

// CommandExecutor class definition
class CommandExecutor {
public:
    void execute(const std::string& command, const std::vector<std::string>& args) {
        if (command == "echo") {
            for (const auto& arg : args) {
                std::cout << arg << " ";
            }
            std::cout << std::endl;
        } else if (command == "help") {
            std::cout << "Supported commands: echo, help, exit\n";
        } else {
            std::cerr << "Unknown command: " << command << std::endl;
        }
    }
};

// CommandParser class definition
class CommandParser {
public:
    std::pair<std::string, std::vector<std::string>> parse(const std::string& input) {
        std::istringstream iss(input);
        std::string command;
        std::vector<std::string> args;
        iss >> command;
        std::string arg;
        while (iss >> arg) {
            args.push_back(arg);
        }
        return {command, args};
    }
};

// Terminal class definition
class Terminal {
private:
    ThreadPool threadPool;
    CommandParser parser;
    CommandExecutor executor;

public:
    Terminal(size_t numThreads) : threadPool(numThreads) {}

    void start() {
        std::string input;
        std::cout << "Terminal started. Type 'help' for commands.\n";
        while (true) {
            std::cout << "> ";
            std::getline(std::cin, input);

            if (input.empty()) continue;

            if (input == "exit") break;

            auto [command, args] = parser.parse(input);

            threadPool.enqueue([this, command, args]() {
                executor.execute(command, args);
            });
        }
        std::cout << "Terminal exiting...\n";
    }
};

int main() {
    Terminal terminal(4); // Initialize with 4 threads in the thread pool
    terminal.start();
    return 0;
}

