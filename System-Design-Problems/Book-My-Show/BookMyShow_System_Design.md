
# BookMyShow System Design

## 1. Understanding Requirements

The system allows users to:
1. Search for movies/events.
2. View theaters or venues and available slots.
3. Book tickets.
4. Handle payments (abstracted here).
5. Manage user accounts and bookings.
6. Handle concurrency for seat reservations to avoid double bookings.

---

## 2. Class Design

### Key Classes:
1. **Movie/Event**: Represents movies or events with showtimes.
2. **Theater**: Represents a venue with screens/auditoriums.
3. **Screen/Auditorium**: Contains seat details for a show.
4. **Seat**: Represents individual seats.
5. **Show**: Represents a specific show of a movie or event.
6. **User**: Represents the user booking the tickets.
7. **Booking**: Represents a ticket booking with seat and show details.
8. **Payment**: Handles payment processing (abstracted).
9. **Thread Pool**: Handles concurrent seat reservations.

---

## 3. Class Diagram

### Textual Representation

#### **Movie**
- **Attributes**:
  - `name: std::string`
  - `genre: std::string`
  - `duration: int`
  - `shows: std::vector<std::shared_ptr<Show>>`
- **Methods**:
  - `Movie(const std::string& name, const std::string& genre, int duration)`
  - `addShow(std::shared_ptr<Show> show)`
  - `getName() const -> const std::string&`
  - `getShows() const -> const std::vector<std::shared_ptr<Show>>&`

#### **Theater**
- **Attributes**:
  - `name: std::string`
  - `location: std::string`
  - `screens: std::vector<std::shared_ptr<Screen>>`
- **Methods**:
  - `Theater(const std::string& name, const std::string& location)`
  - `addScreen(std::shared_ptr<Screen> screen)`
  - `getScreens() const -> const std::vector<std::shared_ptr<Screen>>&`

#### **Screen**
- **Attributes**:
  - `screenId: int`
  - `rows: int`
  - `cols: int`
  - `seats: std::vector<std::vector<std::shared_ptr<Seat>>>`
  - `show: std::shared_ptr<Show>`
- **Methods**:
  - `Screen(int screenId, int rows, int cols)`
  - `reserveSeat(int row, int col) -> bool`
  - `assignShow(std::shared_ptr<Show> show)`
  - `displaySeats() const`

#### **Seat**
- **Attributes**:
  - `isReserved: bool`
  - `mtx: std::mutex`
- **Methods**:
  - `Seat()`
  - `reserve() -> bool`
  - `isAvailable() const -> bool`

#### **Show**
- **Attributes**:
  - `movie: std::shared_ptr<Movie>`
  - `timeSlot: std::string`
  - `screen: std::shared_ptr<Screen>`
- **Methods**:
  - `Show(std::shared_ptr<Movie> movie, const std::string& timeSlot, std::shared_ptr<Screen> screen)`
  - `getMovie() const -> std::shared_ptr<Movie>`
  - `getScreen() const -> std::shared_ptr<Screen>`
  - `getTimeSlot() const -> const std::string&`

#### **User**
- **Attributes**:
  - `name: std::string`
  - `bookingHistory: std::vector<std::shared_ptr<Booking>>`
- **Methods**:
  - `User(const std::string& name)`
  - `addBooking(std::shared_ptr<Booking> booking)`
  - `getBookingHistory() const -> const std::vector<std::shared_ptr<Booking>>&`

#### **Booking**
- **Attributes**:
  - `user: std::shared_ptr<User>`
  - `show: std::shared_ptr<Show>`
  - `seat: std::pair<int, int>` (row, column)
- **Methods**:
  - `Booking(std::shared_ptr<User> user, std::shared_ptr<Show> show, std::pair<int, int> seat)`
  - `getDetails() const`

#### **BookingService**
- **Attributes**:
  - `threadPool: ThreadPool`
- **Methods**:
  - `BookingService(size_t threads)`
  - `bookTicket(std::shared_ptr<User> user, std::shared_ptr<Show> show, int row, int col)`

#### **ThreadPool**
- **Attributes**:
  - `workers: std::vector<std::thread>`
  - `tasks: std::queue<std::function<void()>>`
  - `mtx: std::mutex`
  - `cv: std::condition_variable`
  - `stop: bool`
- **Methods**:
  - `ThreadPool(size_t threads)`
  - `~ThreadPool()`
  - `enqueue(F&& task)`

---

## 4. Interaction Workflow

### **Search Flow**
1. User queries for movies or events.
2. System retrieves relevant movies and available shows.

### **Booking Flow**
1. User selects a show and seats.
2. System checks seat availability (concurrent-safe).
3. If seats are available, the system reserves them and processes payment.

---

## 5. Code Implementation

### **File Structure**
- `Movie.h` – Represents movies or events.
```
#pragma once
#include <string>
#include <vector>
#include <memory>

class Show; // Forward Declaration

class Movie 
{
    std::string name;
    std::string genre;
    int duration; // in minutes
    std::vector<std::shared_ptr<Show>> shows;
    
public:
    Movie(const std::string& name, const std::string& genre, int duration)
        : name(name), genre(genre), duration(duration) {}

    void addShow(std::shared_ptr<Show> show);
    const std::string& getName() const { return name; }
    const std::vector<std::shared_ptr<Show>>& getShows() const { return shows; }
};

```

- `Theater.h` – Represents theaters/venues.
```
#pragma once
#include <string>
#include <vector>
#include <memory>

class Screen;

class Theater {
    std::string name;
    std::string location;
    std::vector<std::shared_ptr<Screen>> screens;
    
public:
    Theater(const std::string& name, const std::string& location)
        : name(name), location(location) {}

    void addScreen(std::shared_ptr<Screen> screen);
    const std::vector<std::shared_ptr<Screen>>& getScreens() const { return screens; }
};

```
- `Screen.h` – Represents screens/auditoriums in theaters.
```
#pragma once
#include <vector>
#include <memory>
#include <mutex>

class Seat;
class Show;

class Screen {
    int screenId;
    int rows, cols;
    std::vector<std::vector<std::shared_ptr<Seat>>> seats;
    std::shared_ptr<Show> show;
    
public:
    Screen(int screenId, int rows, int cols);
    bool reserveSeat(int row, int col);
    void assignShow(std::shared_ptr<Show> show);
    void displaySeats() const;
};
```
- `Seat.h` – Represents individual seats.
```
#pragma once

#include <mutex>

class Seat {
    bool isReserved;
    std::mutex mtx;

public:
    Seat() : isReserved(false) {}
    bool reserve();
    bool isAvailable() const;
};
```

- `Show.h` – Represents showtimes.
```
#pragma once
#include <string>
#include <memory>
#include <vector>

class Movie;
class Screen;

class Show 
{
    std::shared_ptr<Movie> movie;
    std::string timeSlot;
    std::shared_ptr<Screen> screen;

public:
    Show(std::shared_ptr<Movie> movie, const std::string& timeSlot, std::shared_ptr<Screen> screen)
        : movie(movie), timeSlot(timeSlot), screen(screen) {}

    std::shared_ptr<Movie> getMovie() const { return movie; }
    std::shared_ptr<Screen> getScreen() const { return screen; }
    const std::string& getTimeSlot() const { return timeSlot; }
};
```

- `ThreadPool.h` – Manages concurrent tasks.
```
#pragma once
#include <thread>
#include <queue>
#include <vector>
#include <mutex>
#include <condition_variable>
#include <functional>

class ThreadPool 
{
    std::vector<std::thread> workers;
    std::queue<std::function<void()>> tasks;
    std::mutex mtx;
    std::condition_variable cv;
    bool stop;

public:
    ThreadPool(size_t threads);
    ~ThreadPool();
    template <class F>
    void enqueue(F&& task);
};
```

- `BookingService.h` – Handles booking operations.

```
#pragma once
#include "User.h"
#include "Show.h"
#include "ThreadPool.h"

class BookingService 
{
    ThreadPool threadPool;

public:
    BookingService(size_t threads) : threadPool(threads) {}
    void bookTicket(std::shared_ptr<User> user, std::shared_ptr<Show> show, int row, int col);
};

```
- `Main.cpp` – Entry point demonstrating usage.

```
#include "Movie.h"
#include "Theater.h"
#include "Screen.h"
#include "Seat.h"
#include "Show.h"
#include "ThreadPool.h"
#include "BookingService.h"

int main() 
{
    auto movie = std::make_shared<Movie>("Inception", "Sci-Fi", 148);

    auto screen1 = std::make_shared<Screen>(1, 5, 5);
    auto screen2 = std::make_shared<Screen>(2, 6, 6);

    auto theater = std::make_shared<Theater>("PVR Cinemas", "NYC");
    theater->addScreen(screen1);
    theater->addScreen(screen2);

    auto show1 = std::make_shared<Show>(movie, "10:00 AM", screen1);
    auto show2 = std::make_shared<Show>(movie, "02:00 PM", screen2);

    movie->addShow(show1);
    movie->addShow(show2);

    auto bookingService = BookingService(4);

    auto user = std::make_shared<User>("John Doe");
    bookingService.bookTicket(user, show1, 2, 3);

    return 0;
}

```
 
---

## 6. Relationships

1. **Movie - Show**: One-to-Many
   - A `Movie` has multiple `Show`s.

2. **Theater - Screen**: One-to-Many
   - A `Theater` contains multiple `Screen`s.

3. **Screen - Seat**: One-to-Many
   - A `Screen` contains multiple `Seat`s arranged in rows and columns.

4. **Show - Movie**: Many-to-One
   - A `Show` is associated with one `Movie`.

5. **Show - Screen**: One-to-One
   - A `Show` is assigned to one `Screen`.

6. **User - Booking**: One-to-Many
   - A `User` can have multiple `Booking`s.

7. **Booking - Show**: One-to-One
   - A `Booking` is linked to one `Show`.

8. **BookingService - ThreadPool**: Association
   - `BookingService` uses `ThreadPool` to handle concurrent requests.
