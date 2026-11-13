# OVERLAYS

This directory contains overlays for VSS. Use of overlays is a new concept for VSS.
The overlays in this folder shall currently be seen as examples only, and are not part of the official VSS specification.

## Profiles

Larger overlays to adapt VSS to a specific vehicle category, like motorbikes.

## Extensions

Smaller overlays typically to be applied after applying profiles (if any).

* `Trailers.vspec` - Proposal for up to 5 dynamically coupled trailers (`Vehicle.Trailers`), using the extended
  attribute `dynamic`. Requires `-e dynamic` when running vss-tools in strict mode.
  See `docs-gen/content/extensions/dynamic_instances.md` and the examples in `extensions/Trailers/examples`.
