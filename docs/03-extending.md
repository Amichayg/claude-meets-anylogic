# Extending the Model: Truck Appointment Scheduling

This guide walks through exactly how we added a truck appointment scheduling system to the grain terminal. You'll learn the pattern for extending any AnyLogic model by editing the `.alp` XML directly.

## What We're Building

**Before:** Trucks arrive randomly and queue in FIFO order. First come, first served.

**After:** Trucks are assigned appointment time slots. On-time trucks get highest priority. Early trucks get moderate priority. Late trucks are deprioritized. Trucks waiting too long are ejected as no-shows.

## The Approach

AnyLogic models are stored as XML in `.alp` files. While you'd normally use the AnyLogic GUI to make changes, understanding the XML gives you the ability to:
- Script bulk changes
- Version control meaningful diffs
- Understand exactly what AnyLogic generates
- Make surgical edits without the GUI

We'll add:
1. New **parameters** (configurable inputs)
2. New **variables** (runtime state)
3. New **functions** (logic)
4. Modified **flowchart block** settings (queue behavior)

## Step 1: Add Parameters to Main

Parameters are inputs you can tune in the experiment UI. We need four:

| Parameter | Type | Default | Purpose |
|-----------|------|---------|---------|
| `appointmentSlotDuration` | double | 300 | Slot length in seconds (5 min) |
| `appointmentWindowTolerance` | double | 60 | Grace period for "on time" |
| `noShowTimeout` | double | 600 | Seconds before queue ejection |
| `maxTrucksPerSlot` | int | 4 | Capacity per slot |

In the XML, parameters live inside `<Variables>` as `<Variable Class="Parameter">`. Here's the pattern:

```xml
<Variable Class="Parameter">
    <Id>1700000000010</Id>                <!-- Unique ID -->
    <Name><![CDATA[appointmentSlotDuration]]></Name>
    <X>-350</X><Y>200</Y>                <!-- Position in the editor -->
    <Label><X>10</X><Y>0</Y></Label>
    <PublicFlag>false</PublicFlag>
    <PresentationFlag>true</PresentationFlag>
    <ShowLabel>true</ShowLabel>
    <Properties SaveInSnapshot="true" ModificatorType="STATIC">
        <Type><![CDATA[double]]></Type>
        <UnitType><![CDATA[NONE]]></UnitType>
        <SdArray>false</SdArray>
        <DefaultValue Class="CodeValue">
            <Code><![CDATA[300]]></Code>   <!-- Default value -->
        </DefaultValue>
        <ParameterEditor>
            <Id>1700000000011</Id>
            <Label><![CDATA[Appointment slot duration [sec]]]></Label>
            <EditorContolType>TEXT_BOX</EditorContolType>
            <MinSliderValue><![CDATA[0]]></MinSliderValue>
            <MaxSliderValue><![CDATA[1000]]></MaxSliderValue>
            <DelimeterType>NO_DELIMETER</DelimeterType>
        </ParameterEditor>
    </Properties>
</Variable>
```

**Key points:**
- Every element needs a unique `<Id>`. We use 170000000XXXX to avoid collisions with existing IDs.
- `<ParameterEditor>` defines how it appears in the AnyLogic experiment UI.
- `ModificatorType="STATIC"` means it's set before the run starts.

We insert these after the existing `trainCapacity` parameter, before the `<Variable Class="CollectionVariable">` section.

## Step 2: Add Tracking Variables to Main

Variables track runtime state. We need counters and a slot tracker:

```xml
<Variable Class="PlainVariable">
    <Id>1700000000001</Id>
    <Name><![CDATA[nextAppointmentTime]]></Name>
    <X>-150</X><Y>275</Y>
    <Label><X>10</X><Y>0</Y></Label>
    <PublicFlag>false</PublicFlag>
    <PresentationFlag>true</PresentationFlag>
    <ShowLabel>true</ShowLabel>
    <Properties SaveInSnapshot="true" Constant="false"
                AccessType="public" StaticVariable="false">
        <Type><![CDATA[double]]></Type>
        <InitialValue Class="CodeValue">
            <Code><![CDATA[0]]></Code>
        </InitialValue>
    </Properties>
</Variable>
```

Variables we added to Main:
- `nextAppointmentTime` (double) — the next slot's start time
- `trucksInCurrentSlot` (int) — how many trucks are booked in the current slot
- `totalOnTime`, `totalLate`, `totalEarly`, `totalNoShow` (int) — KPI counters

## Step 3: Add Variables to the Truck Agent

Each truck needs to know its appointment details. Find the `<ActiveObjectClass>` with `<Name>Truck</Name>` and add to its `<Variables>`:

| Variable | Type | Default | Purpose |
|----------|------|---------|---------|
| `appointmentTime` | double | 0 | Scheduled slot time |
| `arrivalTime` | double | 0 | When truck actually arrived |
| `appointmentPriority` | double | 5.0 | Queue priority score |
| `punctuality` | int | 0 | 0=on-time, 1=early, 2=late |

Same XML pattern as Main variables, just inside the Truck agent's `<Variables>` block.

## Step 4: Add the Assignment Function

Functions in AnyLogic are Java methods. They live inside `<Functions>` in the agent's XML. Here's our core function:

```xml
<Function AccessType="default" StaticFunction="false">
    <ReturnModificator>VOID</ReturnModificator>
    <ReturnType><![CDATA[double]]></ReturnType>
    <Id>1700000000020</Id>
    <Name><![CDATA[assignAppointment]]></Name>
    <X>-550</X><Y>550</Y>
    <Label><X>10</X><Y>0</Y></Label>
    <PublicFlag>false</PublicFlag>
    <PresentationFlag>true</PresentationFlag>
    <ShowLabel>true</ShowLabel>
    <Parameter>
        <Name><![CDATA[truck]]></Name>
        <Type><![CDATA[Truck]]></Type>
    </Parameter>
    <Body><![CDATA[
// Assign the next available appointment slot
double now = time();

// Advance slot if it's in the past
if (nextAppointmentTime < now) {
    nextAppointmentTime = Math.ceil(now / appointmentSlotDuration)
                          * appointmentSlotDuration;
    trucksInCurrentSlot = 0;
}

// Move to next slot if current is full
if (trucksInCurrentSlot >= maxTrucksPerSlot) {
    nextAppointmentTime += appointmentSlotDuration;
    trucksInCurrentSlot = 0;
}

truck.appointmentTime = nextAppointmentTime;
truck.arrivalTime = now;
trucksInCurrentSlot++;

// Compute punctuality-based priority
double diff = now - truck.appointmentTime;
if (diff < -appointmentWindowTolerance) {
    truck.punctuality = 1; // EARLY
    truck.appointmentPriority = 5.0;
    totalEarly++;
} else if (diff <= appointmentWindowTolerance) {
    truck.punctuality = 0; // ON_TIME
    truck.appointmentPriority = 10.0;
    totalOnTime++;
} else {
    truck.punctuality = 2; // LATE
    truck.appointmentPriority = Math.max(1.0,
        10.0 - (diff / appointmentSlotDuration) * 5.0);
    totalLate++;
}
    ]]></Body>
</Function>
```

**The logic:**
- Time is divided into 5-minute slots, each holding up to 4 trucks
- When a slot fills, we advance to the next one
- Priority: on-time = **10**, early = **5**, late = degrades from 10 down to **1** based on how late

## Step 5: Hook It Into the Process

### 5a. Call `assignAppointment` when trucks enter

Find the `carSource` block (the Road Traffic library block that spawns trucks). Its `onExit` parameter was empty. We add:

```xml
<Parameter>
    <Name><![CDATA[onExit]]></Name>
    <Value Class="CodeValue">
        <Code><![CDATA[assignAppointment( (Truck)car );]]></Code>
    </Value>
</Parameter>
```

Now every truck gets an appointment the moment it enters the system.

### 5b. Set queue priority on `seizeRow1`

Find the `seizeRow1` Seize block. Its `priority` parameter was empty. We set it:

```xml
<Parameter>
    <Name><![CDATA[priority]]></Name>
    <Value Class="CodeValue">
        <Code><![CDATA[agent.appointmentPriority]]></Code>
    </Value>
</Parameter>
```

Higher-priority trucks (on-time = 10) get resources first.

### 5c. Change `exitQueue` ordering

The exit queue previously used `QUEUING_COMPARISON` with a custom function. We switch to priority-based ordering:

```xml
<Parameter>
    <Name><![CDATA[queuing]]></Name>
    <Value Class="CodeValue">
        <Code><![CDATA[self.QUEUING_PRIORITY]]></Code>
    </Value>
</Parameter>
<Parameter>
    <Name><![CDATA[priority]]></Name>
    <Value Class="CodeValue">
        <Code><![CDATA[agent.appointmentPriority]]></Code>
    </Value>
</Parameter>
```

### 5d. Enable no-show timeout on `seizeRow1`

Enable the timeout and use our `noShowTimeout` parameter:

```xml
<Parameter>
    <Name><![CDATA[enableTimeout]]></Name>
    <Value Class="CodeValue">
        <Code><![CDATA[true]]></Code>
    </Value>
</Parameter>
<Parameter>
    <Name><![CDATA[timeout]]></Name>
    <Value Class="CodeUnitValue">
        <Code><![CDATA[noShowTimeout]]></Code>
        <Unit Class="TimeUnits"><![CDATA[SECOND]]></Unit>
    </Value>
</Parameter>
<Parameter>
    <Name><![CDATA[onExitTimeout]]></Name>
    <Value Class="CodeValue">
        <Code><![CDATA[totalNoShow++;
releaseParkingPlace(agent);]]></Code>
    </Value>
</Parameter>
```

Trucks waiting over 600 seconds are ejected, their parking spot is freed, and the no-show counter increments.

## Step 6: Validate

After editing the XML, check that it's still valid:

```bash
xmllint --noout "model/Grain Terminal1.alp"
```

Then open it in AnyLogic to verify everything compiles and runs.

## The Pattern

Any extension to an AnyLogic model follows this pattern:

1. **Add parameters** for tunable inputs
2. **Add variables** to entity agents for per-entity state
3. **Add variables** to Main for global tracking
4. **Write functions** in Main for the core logic
5. **Wire it in** by modifying existing flowchart block parameters (onEnter, onExit, priority, conditions)
6. **Validate** XML and test in AnyLogic

The XML structure is predictable once you've seen it. Every block, variable, and function follows a consistent template.

## Ideas for Further Extension

Try these on your own:

- **Grain quality inspection** — add a `SelectOutput` after truck arrival with a rejection probability. Rejected trucks leave immediately.
- **Maintenance downtime** — add a cyclic event that temporarily reduces silo row capacity (set `rows` resource pool count).
- **Weather delays** — add a state variable `weatherOK` and gate ship arrivals on it. Toggle it with a random event.
- **Dynamic pricing** — track queue length and adjust appointment slot capacity based on demand.

Next: [Three.js Visualization →](04-visualization.md)
