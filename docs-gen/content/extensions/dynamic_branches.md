---
title: "Dynamic Branches (Proposal)"
weight: 8
---

> **Status: Proposal.** This page describes a proposed extension of the VSS language.
> It is implemented with the extended attributes `dynamic`, `events` and `actions` and demonstrated by the overlay
> [overlays/extensions/Trailers.vspec](https://github.com/COVESA/vehicle_signal_specification/blob/master/overlays/extensions/Trailers.vspec).

## Problem

A VSS tree is static. All signals exist for the whole lifetime of the vehicle. Some parts of a vehicle combination change
at runtime, and their structure is not known when the vehicle is built. Trailers are the main example:

* A truck can tow from zero up to several trailers (truck and trailer, B-double, road train with dollies).
* Trailers are coupled and decoupled during operation.
* Trailers come from different manufacturers. Each trailer has its own configuration, for example 1 to 15 axles
  and 1 to 15 wheels per axle.

Modeling the worst case as static signals gives more than 10000 signals under `Vehicle.Trailers`, most of which never exist.

## Concept

A **dynamic branch** is a normal VSS `branch` that additionally declares `events` and `actions`.

* Signals in the branch that are **static** always exist.
* Signals in the branch that are marked **dynamic** (`dynamic: true`) exist only while the branch is resolved.
* An **event** says when something happens, and names a **trigger** signal.
* An **action** says what shall be done, and names the signals it needs as input.
* When an event fires, the **broker** executes the actions listed in the event.

The broker is responsible for watching the trigger signals of all dynamic branches and for executing the actions.
Clients only see the result: dynamic signals appear and disappear in the tree of the broker.

The definition is only an upper bound, similar to `instances`. The static part of the specification
declares what *may* exist, and the resolved branch narrows it to what really exists.

## Extended attributes

The keywords are extended attributes. Tools must whitelist them, for example with
`vspec export json -e dynamic -e events -e actions ...`

### `dynamic`

Marks a signal or branch as dynamic.

| Value  | Meaning |
|--------|---------|
| `true` | The node is created by a resolve action and removed by the action `RemoveDynamicSignals`. |
| absent | The node is static. It is never created or removed by the broker. |

Every dynamic node, branch or signal, shall be marked individually. The broker removes exactly the nodes marked
`dynamic: true`, so it does not need to know the structure of the branch.
Instances created from a branch with `instances` inherit the marker. Nodes in a resolved configuration keep it.

### `actions`

A mapping on a dynamic branch. The key is the action name, and each value defines the action:

| Key           | Required | Meaning |
|---------------|----------|---------|
| `description` | yes      | What the action does. |
| *input keys*  | per action | Full signal names of the inputs, see the actions below. |

An action name shall be one of the actions defined on this page, or an action defined by a profile.
An action is executed on the branch that declares it. Relative references inside the action apply to this branch.

### `events`

A mapping on a dynamic branch. The key is the event name, and each value defines the event:

| Key           | Required | Meaning |
|---------------|----------|---------|
| `description` | yes      | When the event fires. |
| `trigger`     | yes      | Mapping with one entry. The key is the trigger condition (`OnFalseToTrue` or `OnTrueToFalse`), the value is the full signal name of the trigger signal, see [Triggers](#triggers). |
| `actions`     | yes      | List of action names declared on the same branch. The broker executes them in the listed order. |

A branch can declare each event name only once.

## Triggers

A trigger is a signal that the broker observes. An event fires when the trigger signal changes in the way the event
describes. In this proposal triggers are signals of type `boolean`. The trigger signal shall be a **static** signal.
Otherwise it could be removed by the action it triggers.

The broker evaluates a trigger when it receives a new value of the signal. It compares the new value with the last value it has seen.
The first value the broker sees after start does not fire an event on its own, unless it is `true` and the branch has not been resolved yet.
In that case the broker handles it like a change from false to true.

## Trigger conditions

| Condition     | Fires when                                          |
|---------------|-----------------------------------------------------|
| `OnFalseToTrue` | The trigger signal changes from `false` to `true`.  |
| `OnTrueToFalse` | The trigger signal changes from `true` to `false`.  |

If a trigger signal gets a value that equals the previous one, no event fires.
If events of several branches fire together, the broker handles them independently, so one branch can not block another.

## Actions

### `ResolveBranchByUri`

Creates the dynamic signals of the branch from a configuration that is retrieved from a URI.

| Key   | Meaning |
|-------|---------|
| `uri` | Full name of a string `attribute` with the URI. The value is a URI template (RFC 6570). The variable `{id}` is replaced by the value of the signal `Id` in the dynamic branch. |

Behavior of the broker:

1. Read the URI attribute and the signal `Id` of the branch. Expand the template.
2. Request the URI. The service may redirect (HTTP 3xx) or answer directly, so the vehicle needs to know only one endpoint.
3. Create the dynamic signals of the branch from the response. The response is a VSS overlay **rooted at the branch**.
   It uses relative paths, so the same document is valid for any trailer position. It narrows dynamic branches to the
   instances that exist, and gives values to attributes.
4. Remove from the declared definition all dynamic nodes that the response does not contain.

### `ResolveBranchBySignalData`

Creates the dynamic signals of the branch from a definition that the source of the data provides itself in a signal.

| Key    | Meaning |
|--------|---------|
| `data` | Full name of a string signal. The value is the branch definition, as a VSS overlay rooted at the branch, in the YAML vspec syntax. |

Behavior of the broker: Read the value of the data signal and create the dynamic signals as in step 3 and 4 of `ResolveBranchByUri`.

### `RemoveDynamicSignals`

Removes the dynamic signals of the branch. It has no input.

Behavior of the broker: Remove all nodes in the branch that are marked `dynamic: true`, including their values.
Subscriptions of clients to removed signals end. Static signals, the actions and the events are kept.

### Common rules for actions

* If an input is not available when the action starts, for example the `Id` is empty or the data signal has no value,
  the broker waits until the input is available, as long as the trigger signal still has the value that fired the event.
  If the trigger signal changes in the meantime, the pending action is dropped.
* If a resolve action fails, for example because the URI can not be reached, the branch keeps no dynamic signals.
  The broker repeats the action later, as long as the trigger signal is still `true`.
* Resolving a branch that is already resolved first removes its dynamic signals (`RemoveDynamicSignals`).
* The broker shall execute the actions of one branch in the order in which the events fired.

## Trailer model

`Vehicle.Trailers` is a normal branch with the dynamic branches `Trailer1` to `Trailer5`.
Trailers are numbered in coupling order. `Trailer1` is the trailer coupled directly to the towing vehicle.

```text
Vehicle.Trailers
├── MaxCount                      attribute
├── ConnectedCount                sensor
├── ConfigurationUri              attribute, URI template, input of ResolveBranchByUri
├── Trailer1                      dynamic branch   ResolveBranchByUri
├── Trailer2                      dynamic branch   ResolveBranchBySignalData (static signal Vss)
├── Trailer3                      dynamic branch   ResolveBranchByUri
├── Trailer4                      dynamic branch   ResolveBranchByUri
└── Trailer5                      dynamic branch   ResolveBranchByUri
    ├── IsConnected               static sensor    TrailerStatic.vspec, trigger of the events
    ├── Id                        static sensor    TrailerStatic.vspec, e.g. VIN
    ├── Vss                       static sensor    Trailer2 only, input of ResolveBranchBySignalData
    ├── Identification.*          dynamic          TrailerDynamic.vspec
    ├── Type, Length, ...         dynamic attributes
    ├── CurrentWeight             dynamic sensor
    ├── Climate[Zone1..2]         dynamic, instances, TargetTemperature actuator
    └── Axle[Row1..15]            dynamic, instances
        └── Wheel[Position1..15]  dynamic, instances
```

All five trailer branches include the same static file, so they share one definition of the static signals:

```yaml
#include Trailers/TrailerStatic.vspec Vehicle.Trailers.Trailer1
```

The gross list `TrailerDynamic.vspec` is **not** included in the trailer branches. Including it five times would build a
very large tree (15 axles with 15 wheels each, per trailer). The dynamic signals are brought in by the resolved
trailer configuration, see [Tooling](#tooling).

### `TrailerDynamic.vspec` is a gross list

`TrailerDynamic.vspec` and the files it includes (`TrailerAxle.vspec`, `TrailerWheel.vspec`) are the **gross list**
of all signals any trailer can have. It is not the description of one real trailer. It is written for the largest
possible trailer:

* up to 15 axles (`Axle` with instances `Row[1,15]`),
* up to 15 wheels per axle (`Wheel` with instances `Position[1,15]`),
* up to 2 climate zones (`Climate` with instances `Zone[1,2]`),
* every signal and attribute that a trailer of any type can provide.

Rules:

* Every node in the gross list is marked `dynamic: true`, so the broker can remove all of them on `RemoveDynamicSignals`.
* A resolved trailer branch is a **subset** of the gross list. The configuration retrieved by `ResolveBranchByUri`
  or provided by `ResolveBranchBySignalData` narrows the instances to those the trailer really has
  (e.g. `Row[1,3]`, `Position[1,4]`) and omits the signals it does not support.
  It shall not add signals or instances that are not in the gross list.
* The gross list defines the upper bounds. Before resolving, or when the branch is not resolved, clients can not rely
  on any dynamic signal. The signals exist only after the resolve action has finished.
* Changes to the gross list, such as new signals or higher limits, are changes to the specification.
  Trailer configurations stay valid when they only use a subset.
* The gross list is a reference and is not part of the built tree. A configuration reuses the leaf files
  `TrailerAxle.vspec` and `TrailerWheel.vspec`, and declares its own instances of `Axle` and `Axle.Wheel`.
  Narrowing the instances of an already declared branch inside the same build layer is not possible with vss-tools,
  so the configuration declares them with the final count directly.

### Declaration of a trailer branch resolved by URI

```yaml
Vehicle.Trailers.Trailer1:
  type: branch
  description: Dynamic branch of the trailer coupled at position 1, directly to the towing vehicle.
  actions:
    ResolveBranchByUri:
      description: Retrieves the trailer configuration from the URI and creates the dynamic signals of this branch from it.
      uri: Vehicle.Trailers.ConfigurationUri
    RemoveDynamicSignals:
      description: Removes all signals marked dynamic from this branch. Static signals are kept.
  events:
    Connected:
      description: The trigger signal changed from false to true, i.e. a trailer was coupled.
      trigger:
        OnFalseToTrue: Vehicle.Trailers.Trailer1.IsConnected
      actions: [ResolveBranchByUri]
    Disconnected:
      description: The trigger signal changed from true to false, i.e. the trailer was decoupled.
      trigger:
        OnTrueToFalse: Vehicle.Trailers.Trailer1.IsConnected
      actions: [RemoveDynamicSignals]
```

`Trailer3`, `Trailer4` and `Trailer5` are declared in the same way.

### Declaration of a trailer branch resolved by signal data

`Trailer2` defines the additional static signal `Vss` and uses it as the input of its resolve action:

```yaml
Vehicle.Trailers.Trailer2:
  type: branch
  actions:
    ResolveBranchBySignalData:
      description: Creates the dynamic signals of this branch from the branch definition contained in the data signal.
      data: Vehicle.Trailers.Trailer2.Vss
    RemoveDynamicSignals:
      description: Removes all signals marked dynamic from this branch. Static signals are kept.
  events:
    Connected:
      trigger:
        OnFalseToTrue: Vehicle.Trailers.Trailer2.IsConnected
      actions: [ResolveBranchBySignalData]
    Disconnected:
      trigger:
        OnTrueToFalse: Vehicle.Trailers.Trailer2.IsConnected
      actions: [RemoveDynamicSignals]

Vehicle.Trailers.Trailer2.Vss:
  datatype: string
  type: sensor
  description: Branch definition of the trailer coupled at position 2, provided by the trailer.
```

### Sequence

1. The trailer is coupled at position *N*. The towing vehicle sets `TrailerN.Id` and then `TrailerN.IsConnected = true`.
   For `Trailer2` the trailer also provides `Trailer2.Vss`.
2. The broker sees the change of `IsConnected` from false to true. The event `Connected` fires.
3. The broker executes the resolve action of the branch. The dynamic signals appear in the tree.
4. While the trailer is coupled, clients read and subscribe to the dynamic signals.
5. The trailer is decoupled. `IsConnected` changes from true to false. The event `Disconnected` fires,
   and the broker executes `RemoveDynamicSignals`. Only the static signals `IsConnected`, `Id` (and `Vss`) remain.

## Tooling

```bash
vspec export json -u spec/units.yaml --strict -e dynamic -e events -e actions \
  -l overlays/extensions/Trailers.vspec \
  -s spec/VehicleSignalSpecification.vspec -o vss_trailers.json

# with the example trailer configuration bound to Trailer1.
# ExampleSemiTrailer.vspec (included by ExampleCombination.vspec) contains all signals of the gross list for
# Trailer1, limited to 3 axles with 4 wheels each, and 2 climate zones.
vspec export json -u spec/units.yaml --strict -e dynamic -e events -e actions \
  -l overlays/extensions/Trailers.vspec \
  -l overlays/extensions/Trailers/examples/ExampleCombination.vspec \
  -s spec/VehicleSignalSpecification.vspec -o vss_trailers_example.json

# the same, as YAML
vspec export yaml -u spec/units.yaml --strict -e dynamic -e events -e actions \
  -l overlays/extensions/Trailers.vspec \
  -l overlays/extensions/Trailers/examples/ExampleCombination.vspec \
  -s spec/VehicleSignalSpecification.vspec -o vss_trailers.yaml
```

Exporters copy the extended attributes of a branch or signal to the exported node, including to expanded instances.
The exported tree contains the static signals of the trailer branches only. The dynamic signals appear in the tree
of a resolved configuration, as in the example above.

## Open points

* Should `dynamic`, `events` and `actions` become core keywords in vss-tools, with validation that the referenced signals exist,
  that triggers are static boolean signals and that all names in `actions` of an event are declared?
* Format and media type of configuration documents (vspec YAML or exported JSON).
* Authentication and trust model for retrieving configurations from URIs, and for the content of the signal `Vss`.
* Additional events (for example `OnChange`) and actions, and a registry for their names.
