# Routing USB 2.0 Differential Pairs on a 4-Layer Board

A practical write-up on how I routed the USB 2.0 D+/D- differential pair on a 4-layer IoT board — stack-up decisions, impedance calculation, and the layout rules that actually matter (and why).

## Why differential pair routing matters for USB 2.0

USB 2.0 High-Speed signaling runs D+ and D- as a **differential pair** at 480 Mbps. The receiver only cares about the *voltage difference* between the two lines, not their individual voltage relative to ground — which is what makes differential signaling noise-resistant in the first place. But that noise immunity only holds if the pair is treated as a matched pair all the way from the driver to the connector. Break the symmetry — mismatched lengths, inconsistent spacing, a via on only one of the two traces — and you reintroduce common-mode noise, which shows up as EMI radiation and, at the receiver, as jitter and eye-diagram closure.

So the entire job of "routing a diff pair" comes down to one goal: **keep both traces electrically identical to each other, end to end.**

## Stack-up

This board uses a standard 4-layer stack-up, 1.6 mm overall thickness, standard FR4:

| Layer | Name | Type |
|---|---|---|
| 1 | Top | Signal |
| 2 | GND | Signal (solid ground plane) |
| 3 | PWR | Signal (power plane) |
| 4 | Bottom | Signal |

![4-layer stack-up in EasyEDA Layer Manager](STACK%20UP.png)

The USB differential pair was routed on the **top layer**, directly referenced to the solid **GND plane** on layer 2 — this makes it a **microstrip** (specifically edge-coupled microstrip, since D+ and D- run side by side on the same layer). Having an unbroken ground plane directly under the pair is what gives you a controlled, predictable return path — which matters more for high-speed signals than almost anything else on the layout.

## Calculating the trace geometry for 90Ω differential impedance

The USB 2.0 spec calls for a **90Ω differential impedance, with ±15% tolerance** per the official spec — though in practice most fabs and designers target a tighter **±10%** window for extra compliance margin, since PCB manufacturing variance alone eats into that budget. Getting there isn't a guess — it's driven by your actual stack-up (dielectric height and constant), and you solve for trace width and spacing using the edge-coupled microstrip equations.

I used the built-in impedance calculator on **[JLCPCB's PCB design platform](https://design.jlcpcb.com?from=umer)** with the stack-up parameters pulled straight from the fab's own material spec — height (h) and dielectric constant (εr) need to match what the manufacturer actually builds, otherwise the calculation is meaningless.

![Differential impedance calculator - edge coupled microstrip, 90Ω target](trace%20width.png)

Parameters used:
- **Trace type:** Edge Coupled Microstrip
- **Target differential impedance (Zd):** 90 Ω
- **Trace thickness (t):** 1 oz/ft²
- **Dielectric height (h):** 0.11 mm
- **Dielectric constant (εr):** 4.29
- **Trace spacing (s):** 8 mil
- **Solved trace width (w):** 6.7033 mil

If you don't have access to a fab-specific calculator, [DigiKey's PCB Trace Impedance Calculator](https://www.digikey.com/en/resources/conversion-calculators/conversion-calculator-pcb-trace-impedance) is a solid general-purpose alternative for sanity-checking numbers — just make sure whatever tool you use lets you plug in your actual copper weight, dielectric height, and εr rather than using default values.

## Schematic — Type-C connector and supporting components

![Type-C connector schematic with USB 2.0 D+/D- and CC lines](type%20c%20connector.png)

A few things worth calling out here:
- **CC1/CC2** get 5.1kΩ pull-down resistors (R21, R22) — this is what tells a USB-C source "I'm a device that can sink current," and is required even on a USB 2.0-only (non-PD) design.
- **VBUS decoupling** — 10µF bulk + 100nF high-frequency caps placed close to the connector on both the connector-side and downstream rails.
- D+ and D- route straight off pins 5/7 (DP1/DN1) toward the MCU/PHY with no other components in the signal path — no series resistors were needed on this design since the driving IC already provides internal source termination, but check your PHY's datasheet, as many discrete implementations still call for 22Ω series resistors near the driver.

## Layout — the actual routing

![Actual routed differential pair on the layout](actual%20layout.png)

This is the routed pair coming off the Type-C connector at the bottom of the board, running up toward the GND-referenced test/via area near the top.

### Rule 1: Length matching

D+ and D- need to arrive at the receiver within a tight length-matching tolerance. Numbers vary a bit by source, so here's how it's generally quoted:

| Source guidance | Length-matching tolerance |
|---|---|
| USB 2.0 spec (general industry reading) | within ~150 mil (3.81 mm) |
| Common tighter design-house practice | under 0.5 mm (~20 mil) |
| High-speed/production layouts (this project) | kept under ~5–7 mil in the tightest sections, well inside spec |

Any length difference between the two traces becomes **skew** — a timing offset between the two signals — and skew converts directly into common-mode noise, because the receiver briefly sees an "unbalanced" signal every time the pair transitions. Since USB 2.0's tolerance (150 mil) is fairly generous compared to USB 3.0/HDMI/DisplayPort, it's forgiving of small mismatches — but it's still good practice to route as tight as the floorplan allows rather than routing to the edge of spec.

In practice this means adding tuned serpentine (accordion) traces on whichever line is shorter, right after the point where the mismatch happens (e.g., after a connector pin offset or a component swap-around).

### Rule 2: Don't place vias in-line, and don't cut the pair apart

This was the most important rule I enforced on this layout — keep D+/D- on the same layer, unbroken, with no vias on the pair itself unless absolutely unavoidable. There's more than one reason for this, and they stack:

1. **Impedance discontinuity.** A via has a completely different parasitic capacitance/inductance profile than a straight trace. Even one via momentarily changes the local impedance away from your carefully calculated 90Ω, causing a reflection at that point. Do it on only one of the two traces (or with slightly different via placement on each) and you get *asymmetric* reflections — which is worse than a symmetric one, because it directly creates common-mode noise instead of just a small differential reflection.
2. **Return path discontinuity.** If a via moves the signal from top layer to bottom layer, the return current on the reference plane has to find a new path too — usually by jumping through a nearby ground via. If there isn't one close by, the return current takes a longer, higher-inductance detour, which shows up as extra EMI and added loop inductance right at the transition.
3. **Broken plane / cut-through.** Routing a slot or cut in the reference plane directly under the pair (common when routing around connector shield tabs or mounting holes) removes the continuous return path the microstrip depends on. The pair may still look fine in 2D, but electrically it's now behaving like two separate transmission lines with a gap in the middle — this is one of the most common causes of USB signal integrity failures that don't show up until compliance testing.
4. **Length/skew mismatch from asymmetric via stubs.** Even when a via genuinely is unavoidable (e.g. routing under a connector shell), if only one of the two traces takes it — or they use different via sizes/drill — you introduce extra length and extra parasitic delay on just one line, reintroducing the skew problem from Rule 1.

The practical takeaway: route the pair fully on one layer from driver to connector wherever the floorplan allows it, and if you absolutely must transition layers, do it with both traces together, using a symmetric via pair, as close to each other as possible, with ground stitching vias nearby for the return path.

![Asymmetric via + plane cut vs symmetric via pair with ground stitching](via-return-path.svg)

## Tools used

- **Design/schematic/layout:** [EasyEDA](https://easyeda.com)
- **Manufacturing & fabrication:** [JLCPCB](https://design.jlcpcb.com?from=umer)
- **Impedance calculation:** JLCPCB's built-in stack-up/impedance calculator, cross-checked against [DigiKey's PCB Trace Impedance Calculator](https://www.digikey.com/en/resources/conversion-calculators/conversion-calculator-pcb-trace-impedance)

## Summary

Routing a USB 2.0 differential pair correctly isn't about following a checklist blindly — it's about protecting one property end to end: **D+ and D- must stay electrically symmetric**. A solid reference plane, a stack-up-matched impedance calculation, tight length matching, and avoiding asymmetric vias/plane cuts on the pair are what actually deliver that in practice.

## References

- [Altium — Routing Requirements for a USB Interface on a 2-Layer PCB](https://resources.altium.com/p/routing-requirements-usb-20-2-layer-pcb)
- [JLCPCB Blog — USB 3.0 Differential Signaling: A Complete PCB Design Guide](https://jlcpcb.com/blog/usb3-differential-signaling-design-guide)
- [Cadence — USB Design Guidelines with OrCAD X](https://resources.pcb.cadence.com/blog/2025-usb-design-guidelines)
- [Cadence System Analysis — Differential Signal Return Paths and Ground Reference Planes](https://resources.system-analysis.cadence.com/blog/msa2021-differential-signal-return-paths-and-ground-reference-planes)
- [EDN — Return Path Discontinuities and EMI: Understand the Relationship](https://www.edn.com/return-path-discontinuities-and-emi-understand-the-relationship/)
- [PCBWay — Differential Pair Routing Guidelines for High-Speed PCB Design](https://www.pcbway.com/blog/PCB_Design_Layout/Differential_Pair_Routing_Guidelines_for_High_Speed_PCB_Design_43465b72.html)
- [Altium — What are Differential Pairs and Differential Signals?](https://resources.altium.com/p/what-are-differential-pairs-and-differential-signals)
- [DigiKey — PCB Trace Impedance Calculator](https://www.digikey.com/en/resources/conversion-calculators/conversion-calculator-pcb-trace-impedance)
