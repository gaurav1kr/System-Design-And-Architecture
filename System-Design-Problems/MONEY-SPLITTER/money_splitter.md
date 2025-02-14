# Money Splitter Application in C++

## **High-Level Design (HLD)**

### **Overview**
The **Money Splitter Application** is a C++ program that helps users split expenses among a group of people. It keeps track of individual contributions and calculates the amount each participant owes or is owed.

### **Features**
1. **User Management**
   - Add/remove users.
   - List all users.
   
2. **Expense Management**
   - Add an expense.
   - Assign the expense to multiple users.
   - Define how the amount is split (equal or custom).

3. **Balance Calculation**
   - Compute who owes whom and how much.
   - Display net balances.

4. **Transaction Settlement**
   - Record settlements when payments are made.
   - Adjust balances accordingly.

5. **Persistence (Optional)**
   - Save and load transactions using a file.

### **System Architecture**
The application follows an **MVC (Model-View-Controller)** approach:
1. **Model:** Represents data structures for Users and Expenses.
2. **Controller:** Manages logic for adding expenses and splitting money.
3. **View:** Displays data and takes user inputs.

---

## **Low-Level Design (LLD)**

### **Class Diagram**
#### **1. Class: `User`**
Represents a user in the system.
```cpp
class User {
private:
    int userId;
    string name;
    double balance; // Positive means they are owed, negative means they owe money.

public:
    User(int id, string name);
    void updateBalance(double amount);
    double getBalance() const;
    string getName() const;
};
```

#### **2. Class: `Expense`**
Represents an expense shared among users.
```cpp
class Expense {
private:
    int expenseId;
    string description;
    double totalAmount;
    int payerId;
    vector<pair<int, double>> participants; // <UserId, AmountOwed>

public:
    Expense(int id, string desc, double amount, int payer, vector<pair<int, double>> users);
    vector<pair<int, double>> getParticipants() const;
    int getPayerId() const;
};
```

#### **3. Class: `MoneySplitter`**
Handles user and expense management.
```cpp
class MoneySplitter {
private:
    unordered_map<int, User> users;
    vector<Expense> expenses;

public:
    void addUser(int id, string name);
    void addExpense(int id, string desc, double amount, int payer, vector<pair<int, double>> participants);
    void showBalances();
    void settleTransactions();
};
```

---

## **Flow of Execution**
1. **Adding Users** → Users are created and stored in a map.
2. **Adding Expenses** → Expenses are stored, and balances are updated.
3. **Showing Balances** → Users can check how much they owe or are owed.
4. **Settling Transactions** → Once payments are made, balances are adjusted.

---

## **Code Implementation (C++)**
Here’s a minimal working implementation:

```cpp
#include <iostream>
#include <unordered_map>
#include <vector>

using namespace std;

// User class
class User {
private:
    int userId;
    string name;
    double balance;

public:
    User(int id, string n) : userId(id), name(n), balance(0) {}

    void updateBalance(double amount) {
        balance += amount;
    }

    double getBalance() const {
        return balance;
    }

    string getName() const {
        return name;
    }
};

// Expense class
class Expense {
private:
    int expenseId;
    string description;
    double totalAmount;
    int payerId;
    vector<pair<int, double>> participants;

public:
    Expense(int id, string desc, double amount, int payer, vector<pair<int, double>> users)
        : expenseId(id), description(desc), totalAmount(amount), payerId(payer), participants(users) {}

    vector<pair<int, double>> getParticipants() const {
        return participants;
    }

    int getPayerId() const {
        return payerId;
    }
};

// MoneySplitter class
class MoneySplitter {
private:
    unordered_map<int, User> users;
    vector<Expense> expenses;

public:
    void addUser(int id, string name) {
        users[id] = User(id, name);
    }

    void addExpense(int id, string desc, double amount, int payer, vector<pair<int, double>> participants) {
        Expense expense(id, desc, amount, payer, participants);
        expenses.push_back(expense);

        for (auto p : participants) {
            users[p.first].updateBalance(-p.second);
        }
        users[payer].updateBalance(amount);
    }

    void showBalances() {
        cout << "\nBalances:\n";
        for (const auto& u : users) {
            cout << u.second.getName() << " has balance: " << u.second.getBalance() << endl;
        }
    }
};

// Main function
int main() {
    MoneySplitter app;

    // Adding users
    app.addUser(1, "Alice");
    app.addUser(2, "Bob");
    app.addUser(3, "Charlie");

    // Adding expenses
    app.addExpense(1, "Dinner", 90, 1, {{2, 30}, {3, 30}}); // Alice paid, Bob & Charlie owe

    // Display balances
    app.showBalances();

    return 0;
}
```

---

## **Future Enhancements**
1. **Graph-based Settlement Optimization**: Minimize transactions required for debt settlement.
2. **Database Integration**: Use SQLite or MySQL for persistent storage.
3. **GUI Implementation**: Convert the console app into a web or desktop app.

---

This document provides a **detailed HLD, LLD, and implementation** for a **Money Splitter Application** in C++. Let me know if you need any modifications! 🚀
