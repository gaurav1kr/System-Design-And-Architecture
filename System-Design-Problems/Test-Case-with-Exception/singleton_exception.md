## High-Level Design (HLD)

### Overview
We are designing a custom exception class in C++ that follows the **Singleton** pattern. This exception class will be used to handle errors in a structured way across an application.

### Components
1. **Singleton Exception Class**
   - Ensures only one instance of the exception class exists.
   - Provides a method to log and throw exceptions.

2. **Test Cases**
   - Verify that only a single instance of the exception class exists.
   - Check exception handling and logging functionalities.

---

## Low-Level Design (LLD)

### Class Design
1. **CustomException Class**
   - Private constructor and destructor.
   - Deleted copy constructor and assignment operator.
   - Static method to get the single instance.
   - Method to throw an exception with a message.
   - Method to log errors.

2. **Main Function**
   - Calls test cases to verify singleton behavior and exception handling.

---

## C++ Implementation

```cpp
#include <iostream>
#include <exception>
#include <string>
#include <mutex>

// Singleton Custom Exception Class
class CustomException : public std::exception {
private:
    std::string errorMessage;
    static CustomException* instance;
    static std::mutex mutex;

    // Private constructor to enforce Singleton
    CustomException() = default;

public:
    // Deleting copy constructor and assignment operator
    CustomException(const CustomException&) = delete;
    CustomException& operator=(const CustomException&) = delete;

    // Static method to get the singleton instance
    static CustomException* getInstance() {
        std::lock_guard<std::mutex> lock(mutex);
        if (!instance) {
            instance = new CustomException();
        }
        return instance;
    }

    // Method to set the error message and throw exception
    void throwException(const std::string& message) {
        this->errorMessage = message;
        throw *this;
    }

    // Override what() method from std::exception
    const char* what() const noexcept override {
        return errorMessage.c_str();
    }

    // Logging error
    void logError(const std::string& message) {
        std::cerr << "[ERROR] " << message << std::endl;
    }
};

// Initialize static members
CustomException* CustomException::instance = nullptr;
std::mutex CustomException::mutex;

// Test function to verify singleton behavior
void testSingleton() {
    CustomException* ex1 = CustomException::getInstance();
    CustomException* ex2 = CustomException::getInstance();
    
    if (ex1 == ex2) {
        std::cout << "Singleton Test Passed: Only one instance exists." << std::endl;
    } else {
        std::cout << "Singleton Test Failed." << std::endl;
    }
}

// Test function to verify exception handling
void testExceptionHandling() {
    try {
        CustomException::getInstance()->throwException("Test Exception");
    } catch (const CustomException& e) {
        std::cout << "Exception Caught: " << e.what() << std::endl;
    }
}

// Main function to run tests
int main() {
    testSingleton();
    testExceptionHandling();
    return 0;
}
```

---

## Key Features in the Implementation
1. **Singleton Pattern**
   - Ensures only one instance of `CustomException` exists.
   - Uses `std::mutex` for thread safety.

2. **Exception Handling**
   - The `throwException()` method sets the error message and throws the exception.
   - The `what()` method provides error details.

3. **Logging Mechanism**
   - The `logError()` method logs messages to `stderr`.

4. **Test Cases**
   - **Singleton Test:** Confirms only one instance is created.
   - **Exception Handling Test:** Ensures exceptions are caught and displayed properly.

---

This approach ensures a structured exception handling system using **Singleton**, which is useful for logging and error management in C++ applications.
