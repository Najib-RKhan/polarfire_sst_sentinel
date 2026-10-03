# Contest

SST Sentinel is entered in **Track 2** of the Microchip / DigiKey PolarFire FPGA Design Contest 2026.

Official page: [PolarFire FPGA Design Contest](https://www.microchip.com/en-us/campaigns/polarfire-fpga-design-contest)

The rules that matter for day-to-day work are summarized below from the PolarFire FPGA Design Contest 2026 official rules. That PDF and the signed-in portal are the controlling text.

## Why Track 2

Each team picks exactly one kit. The kit defines the track.

| Track | Kit | What it is for |
| --- | --- | --- |
| 1 | PolarFire SoC Discovery Kit | Small embedded designs, sensors, light Linux, camera |
| **2** | **PolarFire SoC Icicle Kit** | **Linux, networking, real-time and connected systems** |
| 3 | PolarFire Splash Kit | Fabric only, no hard CPU; high-speed I/O |

Track 2 is the Icicle Kit: five RISC-V cores, 254K logic elements, LPDDR4, eMMC, PCIe, dual Ethernet, and expansion headers. The contest description calls this the track for Linux-based embedded systems and connected real-time work. That matches Sentinel: fabric does the microsecond protection, and the microprocessor subsystem runs bare-metal plus Linux so events can be stored and viewed over Ethernet.

The part named in the spec is **MPFS250T-FCVG484E**. Confirm the exact board revision that arrives before any pin constraints or PCB order.

Selected teams are expected to build on the kit Microchip ships. Extra hardware (the analog front end, gate driver, SiC half-bridge, probes) is the team's cost and risk. The kit alone is not the power stage.

## Schedule

The PDF and the public contest page do not agree on the proposal close. Plan from the earlier date until someone checks the signed-in portal.

| Phase | Rules PDF | Public contest page (fetched 3 Oct 2026) |
| --- | --- | --- |
| Proposal | 7 Aug 2026 – **3 Oct 2026, 11:59 PM PDT** | 7 Aug 2026 – **24 Oct 2026** |
| Selection | 5 Oct – 28 Nov 2026 | 26 Oct – 28 Nov 2026 |
| Build | 30 Nov 2026 – **27 Mar 2027, 11:59 PM PDT** | 30 Nov 2026 – 27 Mar 2027 |
| Winners | 15 May 2027 | 15 May 2027 |

Top 25 teams per track become semi-finalists and may receive a kit. Finalists need a working prototype on that kit.

## What gets submitted

**Proposal (portal form plus one public shared link):**

- Project overview, max 300 words
- Innovation and expected impact, max 250 words
- Track / kit selection
- System block diagram PDF, shared through a public link on an allowed host

The spec's word counts for the overview and innovation text were 271 and 219. The portal counter is what judges see.

**Final package, due 27 Mar 2027:**

- Project documentation, **max 10 pages** (the internal spec is separate and can be long)
- Source code and project files
- README with build and programming instructions
- 3–5 minute demo video
- Disclosure of third-party IP, open-source software, and material AI use

SS-001 is the internal implementation spec. It is not the 10-page contest report.

After a deadline, submitted files in the public folder stay frozen unless Microchip authorizes a change. Later engineering work stays in a different place.

## How it is scored

Proposal: feasibility 35%, innovation 25%, clarity 20%, impact 20%.

Final: code and implementation 35%, innovation 25%, video 20%, documentation 20%.

Implementation scoring explicitly includes RTL, embedded software, use of the kit, and evidence of test. The video and the 10-page report together are 40% of the final score. The spec's demo order is: show a benign event, a comparable fault, an ambiguous event, and what happens when software fails. The dashboard explains the hardware record. It is not the thing being judged as the product.

Prizes per track: winner $2,000, runner-up $1,000, split across the team. One extra $1,000 for best innovation across all tracks. A team can win its track and that extra prize.

## Rules that change how we work

- Team of 2 or 3. One team per person. Team changes after registration need written approval. Everyone must be 18+ and eligible; a long list of countries is excluded in the PDF.
- English only.
- AI tools are allowed if the team understands and owns the work, names the tool, and describes what it did. Unmodified AI output without engineering validation can be disqualified. The spec already discloses ChatGPT/Codex for review, requirements, and document drafting. Update that disclosure to match whatever actually gets built.
- Submissions are licensed to Microchip and DigiKey for broad use, and they are **not confidential**. Do not put secrets in the contest folder.
- A functional prototype on the designated kit, plus enough documentation to explain it, is the minimum bar for a prize.
- Questions go to fpgacontesthelp@microchip.com. The team lead is the point of contact.
