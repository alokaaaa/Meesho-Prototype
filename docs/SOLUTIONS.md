# How each prototype works

This file lists the rules, formulas and thresholds built into each tab of the [live demo](https://incredible-cucurucho-909787.netlify.app/). Impacts are modelled estimates in percentage points (pp) of blended RTO. The levers overlap, so they don't simply add up.

- [1 · Pre-delivery confirm](#1--pre-delivery-confirm-availability-check--in-app-reschedule)
- [2 · DIGIPIN at checkout](#2--digipin-captured-at-checkout-plus-geocode-routing)
- [3 · CoD risk score](#3--cod-risk-score-evidence-not-a-ratio)
- [4 · DIGIPIN gate](#4--digipin-check-at-attempt-failed)
- [5 · Kirana holding nodes](#5--kirana-holding-nodes)
- [6 · Address change](#6--address-change-bounded-by-voronoi-cells)

---

## 1 · Pre-delivery confirm: availability check + in-app reschedule

**Targets:** customer not reachable (30% of RTO) · **Impact:** −0.60 pp

**Trigger.** The message goes out when the parcel is scanned at the last-mile hub (LMDC), and only when **any 2 of these 5 flags** fire:

1. First delivery to this address
2. No house number
3. Worst-quartile pincode
4. CoD over 2× the buyer's usual basket
5. Hub more than 5 km away

About 30% of orders qualify.

**Flow.** A WhatsApp in Hindi asks whether the buyer will be available in the planned slot. She can reply **yes**, or open the Meesho app to **reschedule**. Replies close at **21:00**, and the **05:00** manifest is built by slot.

**Rules built in**

| Rule | Detail |
|------|--------|
| 70% slot cap | Only slots with spare capacity show in the app |
| Both or neither | Two parcels for one buyer are rescheduled together |
| One reschedule | Up to 3 days, held at the hub, with no reverse cost |
| No reply = no change | The parcel goes out as planned; the message costs ₹0.20 |
| Address change in the app only | No message prompts it (see solution 6) |

**Modelled reply mix:** 27% "yes, I'll be there" · 15% reschedule in the app · 58% no reply.

---

## 2 · DIGIPIN captured at checkout, plus geocode routing

**Targets:** address unclear / wrong hub (27% of RTO) · **Impact:** −2.20 pp

At checkout the buyer says who the order is for:

| Path | Share | Address error |
|------|------:|:-------------:|
| Self: one draggable map pin, no login, DIGIPIN saved to the order | 72.0% | 35% → 6% |
| Someone else: the recipient gets a WhatsApp and drops their own pin | 15.4% | 35% → 7% |
| Someone else, no reply in 12 hours: falls back to the saved address | 12.6% | 35% → 14% |
| **Blended** | **100%** | **35% → 7.2%** |

**One-way by design.** The recipient sees who is sending the parcel and can decline. The buyer never sees the returned location, only "Address confirmed".

**Routing.** At destination sort, the DIGIPIN picks the hub, not the typed pincode. This fixes the "wrong hub" half of the problem.

---

## 3 · CoD risk score: evidence, not a ratio

**Targets:** CoD refusal (25% of RTO) · **Impact:** −0.69 pp

```
s = (refusals + α) ÷ (total CoD orders + k)      α = 1.06,  k = 20,  prior m = 5.3%
```

- A buyer is **never scored with fewer than 5 real CoD orders**.
- Older refusals fade with a **12-month half-life**.
- Damaged or wrong items, and refusals the buyer denies, **never count**.

**Tiers**

| Tier | Score | What the buyer sees |
|------|-------|---------------------|
| T1 | < 12% | Nothing changes (~94% of CoD buyers) |
| T2 | 12–20% | ₹25 off to pay online |
| T3 | 20–25% | CoD capped at ₹500 |
| T4 | ≥ 25% | CoD not offered (~1.8% of CoD buyers) |

**The way back.** Three prepaid deliveries move a T4 buyer to T3, with CoD available on a trial basis. Five trial CoD orders with at most one refusal trigger a fresh score.

**Why 25%.** Break-even = (70% churn × ₹400) ÷ (8 CoD orders × ₹170) = **20.6%**. Setting the cut-off at 25% gives a 1.2× margin.

---

## 4 · DIGIPIN check at "attempt failed"

**Targets:** no real attempt (13% of RTO) · **Impact:** −0.92 pp

When a rider taps any failure code, their location is converted to a DIGIPIN and compared with the delivery DIGIPIN:

```
distance ≤ max(limit, 2 × GPS accuracy)
limit = 150 m (dense urban) · 200 m (semi-urban) · 250 m (rural)
```

Inside the limit, the failure is accepted. Outside it, the parcel stays live. The rider gets no penalty either way.

The server also rejects a check when GPS accuracy is worse than 100 m, when a mock-location app is on, or when the rider has moved at an impossible speed since the last scan.

**Three guards against the relabelling trap**

1. **Confirm with the buyer:** a message goes out within 10 minutes. If the buyer denies the failure, the parcel reopens and the rider is flagged.
2. **Benchmark against peers:** a rider more than 1.5σ from the cluster median triggers an audit, not a penalty.
3. **Make the code costly:** "Refused" needs a photo and a reason from a fixed list, with no free text.

---

## 5 · Kirana holding nodes

**Targets:** customer not reachable, Red lane only · **Impact:** −0.65 pp

Red-lane parcels are pulled out at the last-mile hub and dropped together at a kirana: 6–8 parcels at ₹8 each. The shop holds them for up to **72 hours**. The buyer walks 300–400 m to collect, then shows an OTP and pays by cash or UPI.

**Where it's deployed:** only where base RTO beats the break-even.

```
Break-even base RTO = (₹8 + ₹12 − ₹21 + 35% × ₹120) ÷ ₹120 = 34.2%
```

| Cohort | Base RTO | Deploy? |
|--------|---------:|:-------:|
| Dense urban (all) | 15% | No |
| Semi-urban, Red lane | 40% | Yes |
| Rural, Red lane | 46% | Yes |

**Partner selection.** Shops are ranked with KANO weights: basic × 1, performance × 1.2, excitement × 1.5. The criteria are storage capacity, operating hours, walkable coverage, CoD reliability, community trust and digital literacy.

**Shop controls in the demo:** hours since drop, a cash float capped at ₹5,000, and a collection rate of at least 65% within 72 hours.

---

## 6 · Address change, bounded by Voronoi cells

**Targets:** address unclear / wrong hub · **Impact:** −0.80 pp

The change is available **only in the app**, and no message prompts it. Each hub's area is a cell on a Voronoi map. The buyer can move the delivery only into a cell that **shares an edge** with the current one. The re-route is then a sideways move between neighbouring hubs, with a known cost.

**Guardrails**

| Guardrail | Detail |
|-----------|--------|
| Adjacent cell only | A move into a non-adjacent cell offers a free cancel instead |
| One change | One per order, never two |
| 9 pm cut-off | The manifest is frozen after that |
| Both or neither | Applies to stops with more than one parcel |

**Behind the screen**

| | Short edge | Long edge |
|--|:--:|:--:|
| Hub separation · distance to boundary | 4 km · 2 km | 16 km · 8 km |
| New address lands in | 0–2 km band | 5–10 km band |
| RTO in that band | 15% | 24% |
| Sideways re-route cost | ₹10 | ₹22 |
| RTO if the change is refused | 85% | 85% |
| **Net saving per changed order** | **₹109** | **₹82** |
