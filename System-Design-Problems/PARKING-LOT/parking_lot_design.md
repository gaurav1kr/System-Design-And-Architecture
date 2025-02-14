# Parking Lot System Design

## High-Level Design (HLD)

### 1. System Overview
A parking lot system manages the entry, exit, and allocation of parking spots for vehicles. It should support:
- Multiple floors
- Different vehicle types (bike, car, truck)
- Ticketing and billing
- Real-time spot availability tracking

### 2. Key Components
- **Parking Lot**: Manages multiple floors and spots.
- **Parking Spot**: Represents a single parking space.
- **Vehicle**: Stores vehicle details.
- **Ticket**: Represents a parking session.
- **Payment**: Handles billing.
- **Entry & Exit**: Manages check-in/check-out.

### 3. High-Level Diagram
```
+--------------------------------------------------+
|                  Parking Lot System             |
+--------------------------------------------------+
| + Entry Gate         + Parking Lot Manager      |
| + Exit Gate          + Spot Allocation Service  |
| + Payment Service    + Ticketing System         |
| + Database (Parking Spots, Vehicles, Payments)  |
+--------------------------------------------------+
```

## Low-Level Design (LLD)

### 1. Class Design

#### Core Classes
1. `ParkingLot`: Manages floors, spots, and availability.
2. `ParkingFloor`: Contains parking spots.
3. `ParkingSpot`: Represents a parking space.
4. `Vehicle`: Abstract class for vehicle details.
5. `Ticket`: Stores entry time, spot, and vehicle details.
6. `Payment`: Handles payments.
7. `EntryGate/ExitGate`: Controls vehicle movement.

#### Class Diagram
```
+-----------------+       +-----------------+      +-----------------+
|  ParkingLot     | 1 --- | ParkingFloor    | 1--- | ParkingSpot     |
+-----------------+       +-----------------+      +-----------------+
| -floors         |       | -spots          |      | -isAvailable    |
| -parkVehicle()  |       | -assignSpot()   |      | -assignVehicle()|
+-----------------+       +-----------------+      +-----------------+

+----------------+       +-----------------+       +-----------------+
|  Vehicle      |       |  Ticket          |       |  Payment        |
+----------------+       +-----------------+       +-----------------+
| -licensePlate |       | -entryTime       |       | -amount         |
| -type         |       | -exitTime        |       | -pay()          |
+----------------+       +-----------------+       +-----------------+
```

## APIs & Services

### 1. Parking Lot Management API
| Endpoint         | Method | Description                     |
|-----------------|--------|---------------------------------|
| `/park`         | `POST` | Assigns a parking spot         |
| `/unpark`       | `POST` | Frees a parking spot           |
| `/status`       | `GET`  | Returns available spots        |
| `/pay`          | `POST` | Processes payment              |

### 2. Spot Allocation Service
- Allocates parking spots based on availability.
- Supports different vehicle types.

### 3. Ticketing System
- Generates and stores tickets upon entry.
- Tracks duration and calculates cost.

## Modern C++ Implementation

The following implementation:
- Uses **smart pointers** (`std::unique_ptr`)
- Uses **STL containers** (`std::vector`, `std::unordered_map`)
- Follows **OOP principles**
- Uses **Factory pattern** for vehicle creation

```cpp
#include <iostream>
#include <memory>
#include <vector>
#include <unordered_map>
#include <ctime>

// Enum for Vehicle Type
enum class VehicleType { Bike, Car, Truck };

// Base Vehicle Class
class Vehicle {
protected:
    std::string licensePlate;
    VehicleType type;

public:
    Vehicle(std::string plate, VehicleType t) : licensePlate(std::move(plate)), type(t) {}
    virtual ~Vehicle() = default;
    virtual void display() const = 0;
    VehicleType getType() const { return type; }
    std::string getLicense() const { return licensePlate; }
};

// Derived Classes for Different Vehicle Types
class Bike : public Vehicle {
public:
    Bike(std::string plate) : Vehicle(std::move(plate), VehicleType::Bike) {}
    void display() const override { std::cout << "Bike: " << licensePlate << std::endl; }
};

class Car : public Vehicle {
public:
    Car(std::string plate) : Vehicle(std::move(plate), VehicleType::Car) {}
    void display() const override { std::cout << "Car: " << licensePlate << std::endl; }
};

class Truck : public Vehicle {
public:
    Truck(std::string plate) : Vehicle(std::move(plate), VehicleType::Truck) {}
    void display() const override { std::cout << "Truck: " << licensePlate << std::endl; }
};

// Factory to Create Vehicles
class VehicleFactory {
public:
    static std::unique_ptr<Vehicle> createVehicle(const std::string& plate, VehicleType type) {
        switch (type) {
            case VehicleType::Bike: return std::make_unique<Bike>(plate);
            case VehicleType::Car: return std::make_unique<Car>(plate);
            case VehicleType::Truck: return std::make_unique<Truck>(plate);
        }
        return nullptr;
    }
};

// Parking Spot
class ParkingSpot {
    bool isAvailable;
    std::unique_ptr<Vehicle> vehicle;

public:
    ParkingSpot() : isAvailable(true), vehicle(nullptr) {}

    bool parkVehicle(std::unique_ptr<Vehicle> v) {
        if (!isAvailable) return false;
        vehicle = std::move(v);
        isAvailable = false;
        return true;
    }

    void removeVehicle() {
        vehicle.reset();
        isAvailable = true;
    }

    bool available() const { return isAvailable; }
};

// Parking Lot
class ParkingLot {
    std::vector<ParkingSpot> spots;
public:
    ParkingLot(int size) : spots(size) {}

    int park(std::unique_ptr<Vehicle> v) {
        for (size_t i = 0; i < spots.size(); ++i) {
            if (spots[i].available()) {
                spots[i].parkVehicle(std::move(v));
                return i;
            }
        }
        return -1;
    }

    bool unpark(int spotIndex) {
        if (spotIndex < 0 || spotIndex >= spots.size()) return false;
        spots[spotIndex].removeVehicle();
        return true;
    }

    void displayStatus() {
        for (size_t i = 0; i < spots.size(); ++i) {
            std::cout << "Spot " << i << ": " << (spots[i].available() ? "Empty" : "Occupied") << std::endl;
        }
    }
};

int main() {
    ParkingLot lot(5);
    auto car = VehicleFactory::createVehicle("ABC123", VehicleType::Car);
    int spot = lot.park(std::move(car));
    std::cout << "Car parked at spot: " << spot << std::endl;
    lot.displayStatus();
    lot.unpark(spot);
    std::cout << "Car removed." << std::endl;
    lot.displayStatus();
    return 0;
}
```
