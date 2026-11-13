---
title: "Dynamic Instances (Proposal)"
weight: 8
---

> **Status: Proposal.** This page describes a proposed extension of the VSS language.
> It is implemented as the extended attribute `dynamic` and demonstrated by the overlay
> [overlays/extensions/Trailers.vspec](https://github.com/COVESA/vehicle_signal_specification/blob/master/overlays/extensions/Trailers.vspec).

## Problem

[Instances](/vehicle_signal_specification/rule_set/instances/) in VSS are static. They are fixed when the
specification is built, and they describe things that exist for the whole lifetime of the vehicle, such as doors, seats or axles.

Some parts of a vehicle combination change at runtime, and their structure is unknown when the vehicle is built.
Trailers are the main example:

* A truck can tow from zero up to several trailers (truck and trailer, B-double, road train with dollies).
* Trailers are coupled and decoupled during operation.
* Trailers come from different manufacturers. Each trailer has its own configuration, for example a
  number of axles from 1 up to 15, and a number of wheels per axle from 1 up to 15.

If you model this with static instances only, you must instantiate the worst case (5 x 15 x 15 wheels). The
expanded tree in the example overlay then has more than 7000 leaves below `Vehicle.Trailers`. Most of these leaves never exist.
Static instances also cannot express which trailer is bound to an instance, or where its configuration is.

## Proposal

The proposal adds the keyword `dynamic` to `branch` nodes that have `instances`. A dynamic branch keeps the existing
syntax and expansion rules. This means existing tools continue to work if they whitelist `dynamic` as an extended attribute (`-e dynamic`).
The keyword adds the following runtime semantics:

1. **The instance list is an upper bound.** `instances: Trailer[1,5]` declares the instances that *may* exist.
   At runtime, any subset of them is present.
2. **Instances that are not present contain no data.** Servers shall report signals of an instance that is not present as not available.
   Servers do not need to allocate storage for these signals.
3. **The structure of an instance can be resolved at runtime** from a configuration document. This document is a
   normal VSS overlay, see [Configuration documents](#configuration-documents).

`dynamic` has two forms.

### Runtime-dynamic branch (long form)

Use the long form for branches where instances appear and disappear while the vehicle is running.
The value is a mapping. All paths are *full signal names*, starting from the root (e.g. `Vehicle.Trailers…`). They
reference signals defined on the dynamic branch with `instantiate: false`, not inside the instances:

| Key             | Required | Meaning |
|-----------------|----------|---------|
| `presence`      | yes      | Boolean signal. `true` = the instance is present. The placeholder `{instance}` is replaced by the instance name. One presence signal exists for each instance, outside the instances. Clients and brokers can therefore subscribe to it before the instance sub-branch is populated. |
| `key`           | yes      | Signal that uniquely identifies the physical entity bound to each instance (for example the VIN). The placeholder `{index}` is replaced by the zero-based position of the instance in the instance list (`Trailer1` → `0`). Because the key is outside the instances, it is available before the instance sub-branch and its configuration are known. A changed value means that another entity is bound to the instance. |
| `configuration` | no       | String attribute, shared by all instances. It holds a URI template (RFC 6570) with the variable `{id}`, which is replaced by the value of `key`. The result is the URI where the configuration document of the bound entity can be retrieved. |

```yaml
Vehicle.Trailers:
  type: branch
  instances: Trailer[1,5]
  dynamic:
    presence: Vehicle.Trailers.Is{instance}Connected   # IsTrailer1Connected ... IsTrailer5Connected
    key: Vehicle.Trailers.Ids[{index}]                 # Ids[0] ... Ids[4]
    configuration: Vehicle.Trailers.ConfigurationUri
  description: Trailers that can be coupled to and decoupled from the vehicle during operation.

Vehicle.Trailers.IsTrailer1Connected:
  datatype: boolean
  type: sensor
  instantiate: false
  description: Indicates if a trailer is coupled at position 1 (directly to the towing vehicle).
               True = Trailer coupled. False = No trailer coupled.

Vehicle.Trailers.Ids:
  datatype: string[]
  type: sensor
  instantiate: false
  description: Unique identifiers of the currently coupled trailers, ordered by coupling position.
```

The presence and key signals must use `instantiate: false`. Otherwise they would be copied into every instance.

### Configuration-dependent branch (short form)

Use the short form `dynamic: true` for dynamic branches below a runtime-dynamic branch. Their instances do not change
while the same entity is bound. They are given by the configuration of the nearest runtime-dynamic ancestor.

```yaml
Axle:
  type: branch
  instances: Row[1,15]
  dynamic: true
  description: Axle signals of the trailer.
```

### Attributes within dynamic instances

Within a runtime-dynamic instance, a node with `type: attribute` is constant *for as long as the instance is bound
to the same key*. Identification attributes are set when the entity is bound, for example by
reading them from the trailer. All other attributes get their values from the configuration document.
Signals that the trailer or the towing vehicle provides at runtime are modeled as `sensor` or `actuator`, as usual.
The presence and key signals are sensors of the dynamic branch, and the configuration URI is an attribute of the dynamic branch.

### Configuration URI

All instances use one configuration URI, typically provided by the OEM of the towing vehicle:

```yaml
Vehicle.Trailers.ConfigurationUri:
  datatype: string
  type: attribute
  instantiate: false
  description: URI template for retrieving the configuration of a trailer by its unique identifier.
  # value e.g. https://vss.tractor-oem.example/v1/trailer-configurations/{id}
```

The service of the towing vehicle OEM maps the trailer identifier to the trailer OEM. It then returns the configuration
in one of two ways: by redirecting to the trailer OEM (HTTP 3xx), or by forwarding the trailer OEM's response.
The vehicle therefore needs only one trusted endpoint, and it does not need to know all trailer OEMs.

## Trailer model

```text
Vehicle.Trailers                                   branch, instances Trailer[1,5], dynamic (long form)
├── MaxCount                                       attribute, not instantiated
├── ConfigurationUri                               attribute, not instantiated, URI template, configuration
├── ConnectedCount                                 sensor, not instantiated
├── Ids                                            sensor string[], not instantiated, key, ordered by position
├── IsTrailer1Connected … IsTrailer5Connected      sensors, not instantiated, presence of each instance
└── Trailer1 … Trailer5                            numbered in coupling order, Trailer1 = closest to towing vehicle
    ├── Identification.Id                          attribute, equals Ids[N-1] (VIN or similar)
    ├── Identification.IdType / Manufacturer       attributes
    ├── Identification.ConfigurationVersion        attribute, for caching
    ├── Type, Length, Width, Height, GrossWeight, AxleCount   attributes from configuration
    ├── CurrentWeight                              sensor
    └── Axle                                       instances Row[1,15], dynamic: true
        └── Row1 … Row15
            ├── WheelCount, IsLiftAxle, IsSteered, WheelDiameter, TireWidth
            ├── IsLifted, Weight
            └── Wheel                              instances Position[1,15], dynamic: true
                └── Position1 … Position15         counted left to right in driving direction
                    ├── Speed
                    ├── Tire.Pressure / IsPressureLow / AirTemperature
                    └── Brake.PadWear / IsBrakesWorn
```

### Referencing the trailer IDs

The trailer IDs are referenced through the non-instantiated signal `Vehicle.Trailers.Ids`, which is the key of the
dynamic instances. Element *N-1* identifies the trailer bound to `Vehicle.Trailers.TrailerN`.
The IDs are available as soon as the trailers are coupled, before the instance sub-branches are populated and
before their configuration is resolved. A client can subscribe to this single signal to detect when trailers are
coupled, decoupled or reordered. After binding, each instance also exposes the ID as the attribute `Identification.Id`.

The instance number is the *position* of the trailer in the combination, not its identity. If the combination changes,
the same physical trailer can appear at another instance. Clients that track a specific trailer shall use its `Id`.

## Configuration documents

A configuration document is a VSS overlay **rooted at the trailer instance**. Its paths are relative and have no
`Vehicle.Trailers.TrailerN` prefix, so the same document is valid at any position. The document:

* narrows each dynamic branch to the instances that exist (`Axle` → `Row[1,3]`, `Axle.Row1.Wheel` → `Position[1,4]`),
* gives values to attributes with `default`,
* narrows `allowed` to the single actual value for enumerated attributes. This keeps the
  catalog rule that `default` shall equal `allowed[0]`.

```yaml
# Returned via https://vss.tractor-oem.example/v1/trailer-configurations/YS2TRAILER0000001
Identification.ConfigurationVersion:
  datatype: string
  type: attribute
  default: '2026-11-01'
  description: Version of the trailer configuration resolved for this instance.

Type:
  datatype: string
  type: attribute
  allowed: ['SEMI_TRAILER']
  default: 'SEMI_TRAILER'
  description: Type of the trailer.

Axle:
  type: branch
  instances: Row[1,3]
  description: Axle signals of the trailer.

Axle.Row1.Wheel:
  type: branch
  instances: Position[1,4]
  description: Wheel signals of the trailer axle.
```

To bind the document, mount it below the instance where the trailer is coupled. This uses the existing include mechanism:

```yaml
#include ExampleSemiTrailer.vspec Vehicle.Trailers.Trailer1
```

### Resolution sequence

1. The trailer is coupled at position *N*. The server reads the trailer ID (for example via ISO 11992 or the trailer
   telematics unit), sets `Vehicle.Trailers.IsTrailerNConnected = true` and `Ids[N-1]`, and updates `ConnectedCount`.
   Clients already subscribed to `IsTrailerNConnected` and `Ids` are notified before the instance sub-branch is populated.
2. The server sets the attributes `TrailerN.Identification.*` from data read from the trailer.
3. The server expands `Vehicle.Trailers.ConfigurationUri` with `{id}` = `Ids[N-1]` and requests the configuration.
   If a cached configuration exists for the Id, the server sends a conditional request with its `ConfigurationVersion`
   (for example as `If-None-Match`) and reuses the cache if the configuration has not changed.
4. The server mounts the document at `Vehicle.Trailers.TrailerN`. Only the instances it selects become available.
   Attributes get the configured values.
5. On decoupling, the server removes the mount, sets `IsTrailerNConnected = false`, and marks all signals of the instance as not available.
   It updates `ConnectedCount` and `Ids`.

If the server cannot resolve the configuration (for example because the vehicle is offline), the instance is still present.
The dynamic branches then fall back to their upper bound, and signals the trailer does not report stay not available.

## Tooling

Whitelist the keyword with `-e dynamic` to validate and export:

```bash
vspec export json -u spec/units.yaml --strict -e dynamic \
  -l overlays/extensions/Trailers.vspec \
  -s spec/VehicleSignalSpecification.vspec -o vss_trailers.json

# with an example trailer bound to Trailer1
vspec export json -u spec/units.yaml --strict -e dynamic \
  -l overlays/extensions/Trailers.vspec \
  -l overlays/extensions/Trailers/examples/ExampleCombination.vspec \
  -s spec/VehicleSignalSpecification.vspec -o vss_trailers_example.json
```

Exporters copy `dynamic` to the expanded instance nodes. Consumers can use it, for example, to generate lazy or optional
data structures instead of fixed arrays. They can also use `--no-expand` and expand dynamic branches at runtime.

## Open points

* Should `dynamic` become a core keyword in vss-tools, with validation that the paths in the long form exist?
* Standardized media type and format (vspec YAML or exported JSON) for configuration documents served by trailer OEMs.
* Authentication and trust model for fetching configurations from OEM URIs.
* Relation to the existing `Vehicle.Trailer.IsConnected` signal. It could be deprecated in favor of `Vehicle.Trailers.ConnectedCount > 0`.
