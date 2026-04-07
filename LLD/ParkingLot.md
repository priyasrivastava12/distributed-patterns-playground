PARKING LOT LLD

![Parking Lot Class Diagram](parkinglot-class-diagram%20(1).png)


Requirements
The parking lot should have multiple levels, each level with a certain number of parking spots.

The parking lot should support different types of vehicles, such as cars, motorcycles, and trucks.

Each parking spot should be able to accommodate a specific type of vehicle.

The system should assign a parking spot to a vehicle upon entry and release it when the vehicle exits.

The system should track the availability of parking spots and provide real-time information to customers.

The system should handle multiple entry and exit points and support concurrent access.


Classes, Interfaces and Enumerations

The ParkingLot class follows the Singleton pattern to ensure only one instance of the parking lot exists. It maintains a list of levels and provides methods to park and unpark vehicles.

The ParkingFloor class represents a level in the parking lot and contains a list of parking spots. It handles parking and unparking of vehicles within the level.

The ParkingSpot class represents an individual parking spot and tracks the availability and the parked vehicle.

The Vehicle class is an abstract base class for different types of vehicles. It is extended by Car, Motorcycle, and Truck classes.

The VehicleSize enum defines the different types of vehicles supported by the parking lot.
Multi-threading is achieved through the use of synchronized keyword on critical sections to ensure thread safety.

The Main class demonstrates the usage of the parking lot system.

Design Patterns Used:

Singleton Pattern: Ensures only one instance of the ParkingLot class.
Factory Pattern (optional extension): Could be used for creating vehicles based on input.
Observer Pattern (optional extension): Could notify customers about available spots.


---

# 1. Core Enums (Always start with this)

Interviewers like seeing enums.

```java
enum VehicleType {
    BIKE, CAR, TRUCK
}

enum SpotType {
    SMALL, MEDIUM, LARGE
}

enum TicketStatus {
    ACTIVE, PAID, LOST
}
```

---

# 2. Vehicle Class

```java
abstract class Vehicle {
    String licenseNumber;
    VehicleType type;

    public Vehicle(String licenseNumber, VehicleType type) {
        this.licenseNumber = licenseNumber;
        this.type = type;
    }

    public VehicleType getType() {
        return type;
    }
}

class Car extends Vehicle {
    public Car(String licenseNumber) {
        super(licenseNumber, VehicleType.CAR);
    }
}
```

Just remember:

```
Vehicle (abstract)
   ↑
 Car / Bike / Truck
```

---

# 3. Parking Spot

```java
class ParkingSpot {

    int spotId;
    SpotType type;
    Vehicle vehicle;

    public ParkingSpot(int id, SpotType type) {
        this.spotId = id;
        this.type = type;
    }

    public boolean isFree() {
        return vehicle == null;
    }

    public void parkVehicle(Vehicle v) {
        this.vehicle = v;
    }

    public void removeVehicle() {
        this.vehicle = null;
    }
}
```

Core logic:

```
ParkingSpot
 - isFree()
 - parkVehicle()
 - removeVehicle()
```

---

# 4. Level (Floor)

```java
class Level {

    int floorNumber;
    List<ParkingSpot> spots;

    public Level(int floorNumber, int spotCount) {
        this.floorNumber = floorNumber;
        spots = new ArrayList<>();

        for(int i=0;i<spotCount;i++) {
            spots.add(new ParkingSpot(i, SpotType.MEDIUM));
        }
    }

    public ParkingSpot findAvailableSpot() {
        for(ParkingSpot spot : spots) {
            if(spot.isFree()) return spot;
        }
        return null;
    }
}
```

Just remember:

```
Level
 - List<ParkingSpot>
 - findAvailableSpot()
```

---

# 5. Ticket

```java
class ParkingTicket {

    String ticketId;
    Vehicle vehicle;
    ParkingSpot spot;
    long entryTime;

    public ParkingTicket(String id, Vehicle vehicle, ParkingSpot spot) {
        this.ticketId = id;
        this.vehicle = vehicle;
        this.spot = spot;
        this.entryTime = System.currentTimeMillis();
    }
}
```

---

# 6. ParkingLot (Singleton)

```java
class ParkingLot {

    private static ParkingLot instance;
    List<Level> levels;

    private ParkingLot() {
        levels = new ArrayList<>();
    }

    public static ParkingLot getInstance() {
        if(instance == null) {
            instance = new ParkingLot();
        }
        return instance;
    }

    public ParkingSpot findSpot(Vehicle vehicle) {
        for(Level level : levels) {
            ParkingSpot spot = level.findAvailableSpot();
            if(spot != null) return spot;
        }
        return null;
    }
}
```

Remember this **Singleton template**.

---

# 7. Parking Flow (Most Important)

This is what interviewers want.

```java
public ParkingTicket parkVehicle(Vehicle vehicle) {

    ParkingSpot spot = findSpot(vehicle);

    if(spot == null) {
        throw new RuntimeException("Parking Full");
    }

    spot.parkVehicle(vehicle);

    return new ParkingTicket(UUID.randomUUID().toString(), vehicle, spot);
}
```

---

# 8. Unpark Vehicle

```java
public void unparkVehicle(ParkingTicket ticket) {

    ParkingSpot spot = ticket.spot;
    spot.removeVehicle();
}
```

---

# 9. Interview Explanation (Very Important)

Say this clearly:

**Entities**

```
ParkingLot
 ├── Levels
 │     ├── ParkingSpots
 │
 ├── Ticket
 ├── Vehicle
```

**Patterns**

```
Singleton → ParkingLot
Strategy → Spot allocation
Factory → Vehicle creation
Observer → Display board
```

These patterns are commonly applied in parking-lot designs. ([scalabees.com][1])

---

# 🧠 Memory Trick (Very Helpful)

Just remember this chain:

```
Vehicle → ParkingSpot → Level → ParkingLot
                 ↓
               Ticket
```

Flow:

```
Vehicle enters
   ↓
Find Spot
   ↓
Assign Spot
   ↓
Generate Ticket
   ↓
Vehicle exits → free spot
```


