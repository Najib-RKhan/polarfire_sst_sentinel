# SST Sentinel

Track 2 entry for the [PolarFire FPGA Design Contest 2026](https://www.microchip.com/en-us/campaigns/polarfire-fpga-design-contest): a 48 V SiC half-bridge whose protection decision and evidence record run on a PolarFire SoC Icicle Kit.

The FPGA fabric decides whether a current transient is safe to ride through, and it does that without waiting on a processor. Software configures the system while it is disarmed, then archives and explains each event after the hardware has already acted.

These notes follow engineering spec SS-001 rev 0.2 (2 Oct 2026) and the PolarFire FPGA Design Contest 2026 official rules. The spec is a proposed baseline. No hardware, timing closure, or campaign results are claimed yet. When these notes and the spec disagree, the spec wins.

## Read this first

- [Contest and what we have to ship](docs/contest.md)
- [System idea and the software boundary](docs/software-boundary.md)
