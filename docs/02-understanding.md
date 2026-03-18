# Understanding the Grain Terminal Model

This guide walks through the entire simulation — what it does, how it's structured, and where the logic lives.

## The Big Picture

A port-side grain terminal that:
1. **Receives** grain from trucks and trains
2. **Stores** grain in silos
3. **Ships** grain out on bulk carriers

```
 Road → Parking → Auto Silos → Conveyors → Main Silos (×16)
                                                  ↓
 Rail → Train Stop ──────────────────────→ Main Silos (×16)
                                                  ↓
                                            Pier Conveyors
                                                  ↓
                                            Piers (×2) → Ships → Sea
```

## The Three Flows

### Flow 1: Trucks

Trucks are the highest-volume input. A `CarSource` block (from the Road Traffic library) generates trucks at **2 per minute** on a road.

Each truck carries:
- **125 tons** of grain (`truckCapacity`)
- A random grain type (0–3, mapped to wheat/corn/barley/soybean)

**The journey:**

1. **Arrive on road** — `carSource` creates the truck, starts it on the road network
2. **Get parking** — `selectOutput1` calls `getParkPlace(truck)` which checks 4 parking lots. If no space → truck is rejected
3. **Wait in queue** — truck enters `exitQueue`, ordered by appointment priority
4. **Gate check** — `exitHold` holds trucks until `anyCanGoToUnload()` returns true
5. **Seize a row** — `seizeRow1` grabs one of 4 auto silo rows. The resource choice condition calls `findRowToUnload(truck, row)` to check if there's a compatible auto silo with space
6. **Unload** — `truckUnloadingGrain` delay block. Grain flows via a `Valve` from the truck's tank into the selected `AutoSilo` at `maxRate/4`
7. **Release & leave** — row is released, parking spot freed, truck drives away

**Key functions:**
- `getParkPlace(truck)` — finds an open parking lot, records the reservation
- `findRowToUnload(truck, row)` — checks if the row has a silo that fits this grain type/amount
- `isAbleToUnload(truck)` — checks all rows for availability
- `anyCanGoToUnload()` — checks if anyone in the queue can proceed
- `releaseParkingPlace(car)` — decrements the parking reservation

### Flow 2: Auto Silos → Main Silos

Auto silos are intermediate buffers — they decouple truck unloading speed from main silo filling.

When an auto silo reaches **90% capacity** (`autoSiloThr`), `rescheduleTrucks()` triggers:

1. Check no other auto silo is currently unloading (only one at a time)
2. Find the first auto silo above threshold
3. Ask the **Scheduler** agent to create `Order` objects — which main silo should receive this grain?
4. Open the `autoValve` and grain flows through `unloadingConveyorsNet` into the 16 main silos

### Flow 3: Trains

A train is injected at startup via `trainSource.inject()`. Trains carry **1,000 tons** with **8 hopper cars**.

**The journey:**

1. **Arrive** — `trainSource` creates the train on the rail network
2. **Move to stop** — `trainMoveToStop` positions the train at the unloading platform
3. **Unload** — `trainUnloadingGrain` delay block. The train calls `unload()` which asks the Scheduler for orders, then opens `trainValve`. Grain flows **directly** into main silos via `leftUnloadEnter`/`rightUnloadEnter` — bypassing auto silos entirely
4. **Depart** — `trainMoveToExit` and `trainDispose`

### Flow 4: Ships

Every **1,300 seconds** (~22 min), the `shipArrival` timer fires and injects ships.

Each ship has **5 bilges**, each holding **2,000 tons**. On creation, `newRequest()` builds a `ShipRequest`:

- For each bilge: pick a grain type (prefers the type with the most unreserved grain in main silos)
- Bilges hold only one grain type each, but different bilges can differ
- `getUnreservedSize(type)` calculates available grain minus pending ship orders

**The journey:**

1. **Arrive** — `shipSource` creates ships
2. **Seize pier** — `seizePier` grabs one of 2 pier berths
3. **Move to pier** — `moveToPier` + restricted area navigation
4. **Load** — silos receive unload orders. Grain flows through pier conveyors into ship bilges. The `completeOrder` event monitors fill levels
5. **Depart** — pier released, ship navigates out, `shipSink` disposes

## The Scheduler

The `Scheduler` agent is the brain that decides where grain goes:

- `schedule(batch, source)` — given a grain batch (from train or auto silo), determines which main silo(s) should receive it
- `amountInSilos(type)` — how much of each grain type is stored
- `amountOfUnloading(type)` — how much is actively being transferred
- `checkUnloading()` — monitors ongoing transfers

## Agent Types

| Agent | Purpose | Key Properties |
|-------|---------|---------------|
| `Main` | Top-level model, contains all flowcharts | All parameters, functions, collections |
| `Truck` | Delivery vehicle entity | capacity, grainType, appointmentTime, priority |
| `Train` | Rail delivery entity | capacity, grainType, trainSilo |
| `Ship` | Bulk carrier entity | 5 bilges, ShipRequest |
| `Silo` | Main storage (extends AbstractSilo) | unloadOrders, storage tank |
| `AutoSilo` | Intermediate truck buffer | currentBulkType, isLoading, isUnloading |
| `AutoSilosRow` | Groups auto silos, manages row access | findSiloToLoad() |
| `Scheduler` | Coordinates grain routing | schedule(), checkUnloading() |
| `Order` | Single grain transfer task | fromSilo, toSilo, amount |
| `ShipRequest` | Ship's grain requirements | types[], sizes[] per bilge |
| `Pier` | Berth with conveyor connections | 5 conveyor belt references |

## Key Parameters

| Parameter | Value | Meaning |
|-----------|-------|---------|
| `siloCapacity` | 5,000 tons | Each main silo's capacity |
| `truckCapacity` | 125 tons | Grain per truck |
| `trainCapacity` | 1,000 tons | Grain per train |
| `bilgeCapacity` | 2,000 tons | Ship bilge capacity |
| `numOfTypes` | 4 | Number of grain types |
| `nRows` | 4 | Auto silo rows |
| `silosPerRow` | 4 | Main silos per row (16 total) |
| `numberOfShips` | 2 | Ships injected per arrival event |
| `appointmentSlotDuration` | 300 sec | Appointment window length |
| `appointmentWindowTolerance` | 60 sec | On-time grace period |
| `noShowTimeout` | 600 sec | Queue ejection timeout |
| `maxTrucksPerSlot` | 4 | Trucks per appointment slot |

## The .alp File Structure

The model is a single XML file. Here's how it's organized:

```xml
<AnyLogicWorkspace>
  <Model>
    <ActiveObjectClasses>
      <ActiveObjectClass>           <!-- Main agent -->
        <Variables>                  <!-- Parameters and variables -->
        <Events>                     <!-- Timers and scheduled events -->
        <Functions>                  <!-- Java functions -->
        <EmbeddedObjects>            <!-- Flowchart blocks (Source, Queue, Seize...) -->
        <Presentation>               <!-- UI shapes, 3D models, view areas -->
      </ActiveObjectClass>
      <ActiveObjectClass>           <!-- Truck agent -->
      <ActiveObjectClass>           <!-- Ship agent -->
      <!-- ... more agents ... -->
    </ActiveObjectClasses>
  </Model>
</AnyLogicWorkspace>
```

Every agent follows the same pattern: variables, events, functions, embedded objects (flowchart blocks), and presentation elements.

Next: [Extending the Model →](03-extending.md)
