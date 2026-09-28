---
title: Virtual Media Boot

iep-number: 26

creation-date: 2026-09-28

status: implementable

authors:

- "@atd9876"

reviewers:

- "@afritzler"
- "@hardikdr"

---

# IEP-26: Virtual Media Boot

## Table of Contents

- [Summary](#summary)
- [Motivation](#motivation)
    - [Goals](#goals)
    - [Non-Goals](#non-goals)
- [Requirements](#requirements)
- [Proposal](#proposal)
    - [BMC Interface](#bmc-interface)
    - [Configuration delivery](#configuration-delivery)
    - [Lifecycle](#lifecycle)
    - [Conditions](#conditions)
    - [Image Proxy](#image-proxy)
- [Out of Scope](#out-of-scope)
- [Security Considerations](#security-considerations)
- [Testing Strategy](#testing-strategy)
- [Alternatives](#alternatives)

## Summary

Metal-operator today drives a single boot path: it sets a one-time Redfish boot override to `Pxe` and the server boots over the network. This proposal adds **virtual media boot** — inserting a bootable artifact (an ISO) into the BMC's virtual media slot and booting from it — as a path the operator can drive, for environments where booting the server over the network is unavailable, unreliable, or undesirable.

The scope of this IEP is deliberately narrow: it defines **how the operator drives the BMC** for a virtual-media boot — inserting the artifact, arming the one-time override, and reliably ejecting across the claim lifecycle — and how outcomes surface as conditions. It does **not** decide *when* a boot should use virtual media: which transport an image boots over is resolved by the [server boot tool](https://github.com/ironcore-dev/metal-operator/issues/1052) from the image's boot identity, and this IEP consumes that decision.

The design builds on [IEP-20](https://github.com/ironcore-dev/enhancements/blob/main/ieps/20-evolve-server-claim.md) (ServerClaim as the boot primitive, conditions for boot/pull outcomes), the [Image V2 proposal](https://github.com/ironcore-dev/enhancements/pull/49) (bootable artifacts as typed OCI media types), and roadmap #120 (the BMC as the single owner of boot/power transitions).

## Motivation

- **No provisioning network required.** Network boot needs the *server's* NIC to reach boot infrastructure (DHCP/PXE or UEFI-HTTP) before the OS is up. Virtual media boot needs only the *BMC* to reach the image proxy — and BMC reachability is already a hard prerequisite of the operator, since no BMC access means no power control. It is a genuine second path into the same room when the provisioning path is down or being re-cabled.
- **Already-supported capability, never driven.** Virtual media is a standard Redfish capability on every vendor the operator integrates (Dell iDRAC, HPE iLO, Lenovo XClarity, Supermicro/Bull via OpenBMC), and the mock BMC already ships `VirtualMedia` collections. The firmware's boot order will *try* the CD source anyway; what the operator lacks is the ability to *back* that source with real media and make the boot deterministic. This IEP adds exactly that.
- **Consistency with the contract evolution.** IEP-20 makes boot/pull outcomes surface as conditions on the claim. Driving virtual media through the same self-healing and condition machinery keeps boot behavior uniform across transports.

### Goals

- The operator can present a resolved bootable artifact to the BMC and boot from it.
- Server configuration (`ignitionSecretRef`, `userDataRef`) is delivered as a mounted config-drive ISO, so a virtual-media boot works even where the server has no route to the control domain.
- Virtual media is inserted, booted from, and reliably ejected across the claim, release, park, and eviction flows, mirroring the existing self-healing for network-boot overrides.
- Unsupported or failed virtual media boots surface on the claim as conditions, consistent with IEP-20.

### Non-Goals

- **Deciding which transport an image boots over.** That is the [server boot tool](https://github.com/ironcore-dev/metal-operator/issues/1052)'s job — it resolves transport and artifact from the image's boot identity, on demand. This IEP consumes the decision and does not introduce a claim field, label, or flag to influence it.
- **Defining the image manifest.** The config method an image supports (ignition, cloud-init) is declared in the manifest as part of the image's boot identity; defining that schema is the [Image V2 proposal](https://github.com/ironcore-dev/enhancements/pull/49). This IEP consumes the declared method to build the config drive (see [Configuration delivery](#configuration-delivery)).
- Changing the image format (that is the Image V2 proposal).
- Replacing or deprecating network boot — both paths coexist.
- Virtual media for purposes other than boot (e.g. firmware-update media).
- Booting a *parked* server into a maintenance/tool image (see [Out of Scope](#out-of-scope)).

## Requirements

The originating issue lists five minimum requirements. This IEP answers three of them and records why the other two are not part of it:

| # | Requirement | Disposition |
|---|---|---|
| R1 | How a server claim expresses the intent to boot via virtual media | **Out of scope.** No claim field. The server boot tool resolves the transport from the image's boot identity on demand; the tool post-dates this issue and now owns the question. |
| R2 | How the resolved bootable artifact (served by the image proxy) is presented to the BMC for insertion | **[BMC Interface](#bmc-interface).** |
| R3 | How virtual media capability is advertised by a server so claims can be scheduled against it | **Not required.** Virtual media is universal across the integrated vendors, like PXE and UEFI-HTTP — there is nothing to advertise or schedule against. |
| R4 | How virtual media is inserted, booted from, and reliably ejected across claim, release, maintenance, and eviction flows, mirroring the existing self-healing for boot overrides | **[Lifecycle](#lifecycle).** ("Maintenance" maps to the `Parked` state — see [Out of Scope](#out-of-scope).) |
| R5 | How unsupported or failed virtual media boots are surfaced, consistent with the conditions-based reporting introduced by IEP-20 | **[Conditions](#conditions).** |

## Proposal

The controller obtains the transport and the resolved artifact from the server boot tool when it is about to boot a bound server. When the transport is virtual media, the controller drives the BMC sequence below; when it is network, today's `Pxe` override path is unchanged. Nothing about the *decision* lives in this IEP — no new claim field, no per-server capability, no transport list. What this IEP defines is everything from "the artifact is chosen" onward.

### BMC Interface

Four methods are added to the existing `BMC` interface in `bmc/bmc.go`:

```go
type BMC interface {
    // ...existing methods...

    // SetVirtualMediaBootOnce sets a one-time boot override to the
    // virtual media (CD/DVD) source.
    SetVirtualMediaBootOnce(ctx context.Context, systemURI string) error

    // MountVirtualMedia inserts the artifact at mediaURL (an HTTP/HTTPS
    // URL reachable by the BMC) into the given virtual media slot.
    MountVirtualMedia(ctx context.Context, systemURI string, mediaURL string, slotID string) error

    // EjectVirtualMedia ejects the media from the given slot.
    EjectVirtualMedia(ctx context.Context, systemURI string, slotID string) error

    // GetVirtualMediaStatus returns the state of all virtual media
    // devices, used to find a free slot at insert time and for eject
    // bookkeeping.
    GetVirtualMediaStatus(ctx context.Context, systemURI string) ([]*schemas.VirtualMedia, error)
}
```

The artifact URL handed to `MountVirtualMedia` points at the image proxy, not the registry — the BMC never authenticates to a registry (see [Image Proxy](#image-proxy)).

The override target differs from network boot: network boot sets `BootSourceOverrideTarget: Pxe`, virtual media sets `Cd` (both with `Enabled: Once`). `Pxe` is the network target, and the firmware resolves it to DHCP/PXE or UEFI-HTTP itself. Virtual media is the only path that also requires a mount action before the power-on.

`GetVirtualMediaStatus` is how the controller finds a free slot at insert time and what remains to eject at release. Each returned device advertises its `MediaTypes` (`CD`/`DVD`, `USBStick`, `Floppy`), which is how the controller picks a CD/DVD-capable slot for the boot artifact and a suitable second device for the config drive (see [Configuration delivery](#configuration-delivery)). The firmware always *tries* every source in its boot order, so the value of a driven mount is not "can attempt" — it is that the controller can *back* the CD source with real media, making the boot deterministic rather than leaving it to chance.

Vendor semantics (established by the POC implementations):

| Vendor | Insert | Eject | Notes |
|---|---|---|---|
| Dell iDRAC | `POST /Systems/{id}/VirtualMedia/{n}/Actions/...InsertMedia` | `...EjectMedia` | System-scoped; numeric slot IDs `1`/`2` (CD0/CD1); `Image` + `Inserted` payload |
| HPE iLO | `POST /Managers/{id}/VirtualMedia/{n}/Actions/...InsertMedia` | `...EjectMedia` | Manager-scoped; `VirtualMedia/2` is CD, `VirtualMedia/1` is USB |
| Lenovo XClarity | `POST /Managers/{id}/VirtualMedia/EXT{n}/Actions/...InsertMedia` | `...EjectMedia` | Manager-scoped; `EXT`-prefixed slot IDs (`EXT1`/`EXT2`) |
| Supermicro / OpenBMC | TBD | TBD | Collection exposed; API shape to be verified |

Slot IDs are not uniform across vendors, so a normalization step (as in the POC's `normalizeVirtualMediaSlotID`) maps the Redfish-reported ID to the vendor-specific name used by the insert/eject calls.

#### Configuration delivery

Network boot delivers server configuration over the network, and there are **two distinct channels**, not one:

- **Ignition** (`ignitionSecretRef`). The Secret holds an actual ignition config, served over HTTP at an `ignition.url` (boot-operator today). Ignition is pointed at that URL and fetches **ignition-format bytes** — the served payload is already in ignition's own format, so this is a plain URL fetch of a config the agent understands.
- **User-data** (`userDataRef`). The Secret (type `metal.ironcore.dev/user-data`) is served by `metaldata`, an ironcore-specific HTTP metadata service — the cloud-metadata-style channel a cloud-init datasource reads.

Both channels require the *server* to make a network call to the control domain — the very route whose absence motivates virtual media boot. When the server and control domains are separated, a config URL the server must call is unreachable, on either channel.

The channels differ in how much has to change for a disk. `metaldata` is a **bespoke** service — a custom `Metadata-Flavor: IronCore Metal` header, ironcore's own `/v1/…` paths, JSON payloads — that no stock agent reads by default, and its wire format is *not* what a config drive carries. Ignition, by contrast, already speaks its own format on the wire, so its config drive is the same bytes relocated from a URL to a disk. In both cases the config drive carries the same source payload written in the agent's **native local datasource format**:

| Channel | Network form | Config-drive form |
|---|---|---|
| Ignition | ignition config served at `ignition.url` | the same ignition config at the location ignition scans (e.g. `/ignition/config.ign` on a labelled volume) — transport swap only |
| User-data | metaldata JSON at `/v1/…` | user-data written into a cloud-init `nocloud` `cidata` volume — *not* a copy of the metaldata response |

The agent picks the drive up without being told a URL; `metaldata`'s protocol is irrelevant to the disk. Which on-disk format to emit is decided by the image's declared config method (below).

The config therefore travels on the same channel as the boot media: the controller materializes the claim's configuration into a **config-drive ISO** and mounts it as a *second* virtual media device, as the POC already does. Two boundaries make this work:

- The config ISO, like the boot artifact, is fetched by the **BMC** from the image proxy — which is on the always-present BMC↔control path, not the server↔control path.
- The server reads the config ISO as a **local block device** (the standard config-drive / `cidata` convention), so it never has to reach the network for its configuration.

The two devices are **not interchangeable slots**, and vendors differ in what the second device may be. The boot artifact needs a CD/DVD-type device (the `Cd` override boots from it); the config drive needs any readable device the server will pick up. The POC establishes the split:

| Vendor | Boot device | Config device |
|---|---|---|
| Dell iDRAC | `VirtualMedia/1` (CD) | `VirtualMedia/2` (CD) |
| Lenovo XClarity | `EXT1` (CD) | `EXT2` (CD) |
| HPE iLO | `VirtualMedia/2` (CD/DVD) | `VirtualMedia/1` (**USB**) — HPE exposes one CD slot and one USB slot, not two CD slots, and numbers them inversely |
| Supermicro / OpenBMC | TBD | TBD |

So the controller cannot assume "two CD slots": it selects each device by the slot's advertised `MediaTypes` reported by `GetVirtualMediaStatus` — a CD/DVD-capable slot for the boot artifact, and the remaining device (CD or USB, per vendor) for the config drive. This is the same per-vendor slot divergence the `slotID` argument and normalization step already exist for.

A claim with no configuration mounts only the boot artifact; a claim carrying `ignitionSecretRef` or `userDataRef` mounts both. The controller owns generating the config ISO from the referenced Secret and staging it on the appropriate device.

The **config method must be paired with the image**: an OS consumes configuration in exactly one way — ignition reads a specific filesystem/label, cloud-init expects a `cidata` volume with its own label and file layout — and mounting the wrong shape yields a server that boots but is never configured. The controller cannot infer this from the OS, so the **image manifest must declare the config method it supports**. The declared method maps one-to-one onto the two channels above:

| Declared config method | Source Secret | Config drive the controller emits |
|---|---|---|
| `ignition` | `ignitionSecretRef` | the ignition config written to the location ignition scans (e.g. `/ignition/config.ign` on a labelled volume) |
| `cloud-init` (`nocloud`) | `userDataRef` | a `nocloud` `cidata` volume (`meta-data` + `user-data`) built from the Secret |

Defining that manifest field is [Image V2](https://github.com/ironcore-dev/enhancements/pull/49)'s responsibility, as part of the image's boot identity; this IEP **consumes** it — the declared method selects the row above and, with it, the volume label, filesystem, and file layout the controller writes. A claim that carries configuration for an image whose manifest declares no config method, or a method the controller cannot produce, fails with a condition (see [Conditions](#conditions)) rather than booting an unconfigured server.

Open points (to resolve during review):

1. **Where the config ISO is built and served.** The POC generated the config drive in boot-operator and exposed it as a URL the BMC fetched. As the boot controllers move into metal-operator (IEP-20), this needs a home — most naturally the image proxy path, so the BMC fetches the config ISO the same way it fetches the boot artifact. The generation trigger and caching are to be settled here.
2. **Named functions vs `SetBootOverride(target, mode)`.** A generic override API where virtual media is just a new `target` value is a natural fit for the [server boot tool](https://github.com/ironcore-dev/metal-operator/issues/1052). This IEP follows the POC's named-function approach for now; if the tool lands the generic form, these methods collapse into it.

### Lifecycle

The server controller owns the full sequence for a virtual-media boot, mirroring how it already drives the `Pxe` override for network boot. All of the following applies only once the boot tool has resolved the transport to virtual media for the bound server:

1. **Claim bound, server reserved, power off.** Obtain the resolved artifact from the boot tool. Arm the one-time `Cd` override, insert the boot artifact (proxy URL) into a CD/DVD-capable slot, and — if the claim carries configuration — insert the config-drive ISO into a second device (CD or USB per vendor, see [Configuration delivery](#configuration-delivery)), then power on. Media is staged only when the controller is about to power the server on, so reconciliation stays idempotent and does not eject/remount on every loop.
2. **Subsequent power-ons follow the boot order.** After the first boot the firmware follows the server's boot order, possibly reordered by the install; a reboot does not go through the operator. The operator is only involved in **power-ons**, and only the first power-on of the claim gets the one-time media override — later power-ons do not. The media stays in the slot until release (step 5): with a disk at the head of the order it is inert, and if an install failed the firmware falls back to the media, which is the desired retry rather than a re-provisioning loop. *How the controller distinguishes the first power-on from later ones is a boot-lifecycle concern shared with network boot and is deferred to the boot tool — see [Out of Scope](#out-of-scope).*
3. **Working, disk booted.** No boot actions. Reconciles see desired state "no media override" matching actual state.
4. **Server powered off but claimed (recovery path).** A server already past its first power-on that is found off while the claim wants it on is powered on plainly — no media staging, no override.
5. **Claim released / evicted / server returning to available.** Eject all inserted media (discovered via `GetVirtualMediaStatus`, slot IDs normalized per vendor). Eject failures are retried on subsequent reconciles; the POC ran this sweep in the `Available` state handler and it is kept here.
6. **Parked.** The `Parked` server state takes the server out of the claim lifecycle: it is powered off and normal boot/power healing is suspended while the metal-maintenance-operator runs out-of-band day-2 work. On entering park, media is ejected (step 5). Booting a parked server into a tool image is not part of this IEP (see [Out of Scope](#out-of-scope)).
7. **Eviction taints.** An `Evict` taint removes the claim; the release path in step 5 ejects. A `NoBind` taint does not touch media.

Self-healing mirrors the existing boot-override healing: on every reconcile of a bound server the controller compares the media it staged against the actual state (`GetVirtualMediaStatus`), re-inserts if they diverge, and ejects if the claim no longer expects media.

### Conditions

Consistent with IEP-20, boot and pull outcomes are surfaced on the **claim** via conditions:

| Condition | Meaning | Set by |
|---|---|---|
| `MediaInserted` | The artifact is inserted in the BMC and the one-time `Cd` override is armed | controller, after the Redfish insert + override succeed |

The trigger for a post-install eject is deliberately **not** a condition here: the BMC cannot distinguish "booting the installer from the CD" from "booted into the installed OS", so no Redfish observation can decide it, and detecting a completed first boot is a general boot-lifecycle concern, not a virtual-media one.

Failure reporting (each replaces a silent failure with a condition, keeping the claim in `Bound` and the server usable for diagnosis):

| Failure | Condition | Notes |
|---|---|---|
| Redfish insert failed, or the firmware rejects the `Cd` override (no usable slot, slot in use, URL not reachable from BMC) | `MediaInserted=False` with vendor error in message | Distinguishes "BMC cannot reach proxy URL" from slot/target errors; a BMC without a usable slot fails here |
| Config-carrying claim, but the image manifest declares no config method (or one the controller cannot produce) | `MediaInserted=False` (reason `UnsupportedConfigMethod`) | The config method is paired with the image via its manifest; see [Configuration delivery](#configuration-delivery) |
| Eject failed on release | `MediaInserted=True` retained, condition message updated | Server returns to available only after eject succeeds, preventing stale media on the next claim |

Image-resolution failures (registry unreachable, image missing, no matching layer) are reported by the boot-tool/pull path per IEP-20, not by this IEP.

Condition semantics follow the existing conventions: last transition time, `ObservedGeneration`-guarded updates, and retry-on-conflict status patches as used elsewhere in the operator.

### Image Proxy

The BMC fetches the artifact from the image proxy, never from the registry directly. Per IEP-20 the boot controllers move into metal-operator and boot-operator is deprecated; the image proxy is the one boot-operator component that survives, moving into the metal-operator repo as a separate DaemonSet following the existing metaldata pattern (`cmd/metaldata/` + `internal/metaldata/`):

- `cmd/imageproxy/` — entrypoint, deployed as a DaemonSet
- `internal/imageproxy/` — ported from boot-operator's `server/imageproxyserver.go` and `internal/oci/`

It is a pure streaming proxy: clients request a layer by digest, the proxy authenticates to the registry (allowlisted registries only, as today) and streams the layer back. It is transport-agnostic — it already serves network-boot artifacts (UKI, kernel/initrd) and adds ISO for virtual media, selected by OCI media type.

Because the proxy runs on every node, the URL handed to the BMC points at a node-local endpoint. **Constraint:** the BMC must be able to route to the cluster's node (or service) network. Servers in one cluster share the same room and fabric, so this is a single cluster-wide property. Environments where the BMC management network is isolated need the proxy exposed through a stable, externally reachable address (e.g. a Service with an external IP or load balancer) — a one-time infrastructure decision per cluster, after which the operator contract is unchanged.

## Out of Scope

These are explicitly *not* part of this IEP, and why:

- **Transport resolution and boot intent (R1).** Which transport an image boots over, and how a claim would express a preference, is resolved by the [server boot tool](https://github.com/ironcore-dev/metal-operator/issues/1052) from the image's boot identity, on demand. The tool post-dates the virtual-media issue and now owns that question. This IEP adds no claim field and consumes the tool's decision.
- **Per-server capability advertisement (R3).** Virtual media is universal across the integrated vendors, exactly like PXE and UEFI-HTTP. There is no capability to detect, no label to set, and nothing to schedule against; a claim binds to any server. The only per-server variation is at drive time — a BMC that rejects the `Cd` override fails with a condition (see [Conditions](#conditions)) — which is a runtime failure, not a scheduling input.
- **First-power-on detection.** Deciding, at a given power-on, whether to arm the media override or follow the boot order is a boot-lifecycle concern shared with network boot. It is deferred to the boot tool rather than solved for virtual media alone.
- **Booting a parked server into a tool image.** Parking hands the server to the metal-maintenance-operator; if a maintenance flow ever needs a media boot, the continuous-override mode belongs to the boot tool and would be consumed here, not defined here. This IEP only *ejects* on entering park.
- **Discovery via virtual media.** The onboarding probe boots the default way (network boot). Delivering the probe itself as virtual media is a follow-up.

## Security Considerations

- **Config drive carries secrets in the clear.** The config-drive ISO materializes the `ignitionSecretRef`/`userDataRef` Secret onto a block device the server reads locally. Like today's network delivery, the payload is not encrypted at rest on the media; it is mounted only while the claim is bound and ejected on release (see [Lifecycle](#lifecycle)), so it is not left resident after the claim ends. Ejection reliability is therefore a security property, not only a correctness one.
- **BMC-fetched URLs, not server-fetched.** Both the boot artifact and the config drive are fetched by the BMC from the image proxy over the existing BMC↔control path; the server never authenticates to a registry or reaches the control domain. This keeps registry credentials off the server and confines trust to the BMC channel the operator already depends on for power control.
- **Allowlisted registries unchanged.** The image proxy continues to authenticate to allowlisted registries only, as today; virtual media adds ISO as a served media type but no new registry-trust surface.
- **No new inbound exposure on the server.** Because config arrives on a local device rather than an HTTP call, virtual media boot removes, rather than adds, a network dependency for the provisioned server.

## Testing Strategy

- **BMC mock.** The mock already ships `VirtualMedia` collections and honours the `Cd` override; unit and envtest coverage exercises insert/eject/override and the self-healing reconcile (staged vs `GetVirtualMediaStatus`) against it.
- **Per-vendor conformance.** The insert/eject shapes and slot/media-type divergences (Dell, HPE, Lenovo, Supermicro/OpenBMC — see [BMC Interface](#bmc-interface)) are validated against real hardware or vendor simulators, since slot IDs and device media types are not uniform.
- **Config-drive formats.** Generation is tested for each declared config method (ignition config at the scanned location; cloud-init `nocloud` `cidata` volume) from the same source Secret, asserting the agent consumes it without a network call.
- **Lifecycle and conditions.** Claim → bound → boot → release/evict/park transitions assert media is ejected and `status` cleared, and that failures surface the expected conditions (`MediaInserted=False`, `UnsupportedConfigMethod`).

## Alternatives

- **Per-claim transport field.** A `spec.transport: VirtualMedia` on the claim was considered and rejected: it duplicates a decision the [server boot tool](https://github.com/ironcore-dev/metal-operator/issues/1052) already derives from the image's boot identity, and would let a claim request a transport its image cannot serve.
- **Per-server capability advertisement.** Detecting and labelling virtual media support per server (for scheduling) was rejected because the capability is universal across the integrated vendors; see [Out of Scope](#out-of-scope).
- **Config over the network (metaldata) with virtual-media boot.** Keeping config delivery on the network while only the boot artifact rides virtual media was rejected: it reintroduces the server↔control dependency virtual media exists to remove (see [Configuration delivery](#configuration-delivery)).
- **A generic `SetBootOverride(target, mode)` now.** Collapsing the four named BMC methods into a single generic override call is attractive but is deferred to align with the boot tool; see the open point in [BMC Interface](#bmc-interface).

