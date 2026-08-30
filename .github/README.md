# kulhanek/openpbs
This repository is a fork of the [CESNET/openpbs](https://github.com/CESNET/openpbs) repository, which itself is a fork of the original [OpenPBS](https://github.com/openpbs/openpbs)  repository. CESNET uses OpenPBS as a batch system in the [MetaCentrum](https://www.metacentrum.cz/en/) supercomputing centre, operated within the Czech [e-INFRA CZ](https://www.e-infra.cz/) consortium.

We use OpenPBS as the main batch system on several small clusters operated at the [National Centre for Biomolecular Research](https://www.ncbr.muni.cz). The changes in this repository reflect our operating environment, which is based on Ubuntu Server.

## Build Instructions
For build instructions, see [kulhanek/openpbs.build](https://github.com/kulhanek/openpbs.build).

## Versions
### kulhanek/openpbs:ubuntu24.04-ltskernel (rolling updates)
* The branch *ubuntu24.04-ltskernel* is based on CESNET/openpbs:release_20240227
* Better systemd support (individual service units), implementation still suboptimal
* Removed a specific treatment of the *gpu_cap* resource from pbs_sched
* New resource flag combinations implemented: **hl**, **hu**, **ho**, **ha** (based on the CESNET work on **l** and *gpu_cap*)

## New Resources Flags

The new comparison flags are intended for host-level resources and therefore must be used together with the **h** flag. The supported combinations are:

| Flags | Resource type | Comparison semantics |
|---|---|---|
| **hl** | `long`, `float`, `size` | The available value must be **greater than or equal to** the requested value (`available >= request`). |
| **hu** | `long`, `float`, `size` | The available value must be **less than or equal to** the requested value (`available <= request`). |
| **ho** | `string`, `string_array` | **ANY / OR** comparison. At least one requested value in a string or a comma-separated list must be present in the available resource string array or string. |
| **ha** | `string_array` | **ALL / AND** comparison. A string or every value requested in a comma-separated list must be present in the available resource string array. |

The `l`, `u`, and `e` comparison modes are non-consumable numeric comparisons: the resource is used for node selection only and its value is not consumed by a running job. The `e` mode is restricted to resources of type `long`. Similarly, `o` and `a` provide non-consumable matching semantics for string resources, with `o` supported for both `string` and `string_array`, while `a` is supported for `string_array`.

Examples:

```text
create resource gpu_mem type=long flag=hl
create resource interconnect_latency type=long flag=hu
create resource gpu_cap type=string_array flag=ho
create resource cpu_flag type=string_array flag=ha
```

* For `hl`, `gpu_mem=16gb` requests a GPU with minimum GPU memory of 16 GB. 
* For `hu`, `interconnect_latency=5` requests a vnode with maximum latency of 5 microseconds. 
* For `ho`, `gpu_cap=sm_80,sm_90` matches a vnode if either `sm_80` or `sm_90` is available.
* For `ha`, `cpu_flag=intel_ppin,cqm,dca` matches only if all requested CPU flags are available on the vnode.
