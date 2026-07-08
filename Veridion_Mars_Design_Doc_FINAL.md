# Autonomous Systems on Mars: Design Document

**Veridion Engineering Challenge #4, Delivery Engineer**
**Author: Catalin-Marian Hrincescu**

---

# Part 1, Problem Analysis

## 1.1 What is the actual problem?

On the surface this is a path-planning question: there is a dust storm ahead, avoid it. I think that reading is wrong, and most of this document follows from why.

The actual problem: **the aircraft must commit, alone and within minutes, to an action based on a signal it cannot fully trust, a detection that may be a false alarm, describing a hazard that may not even affect this airframe, while every minute of caution costs mission value, a wrong commitment can destroy mission and vehicle together, and no confirmation will ever arrive telling it whether the choice was right.**

Three properties make this the real problem, rather than the storm itself:

- **The decision is forced and time-boxed.** "Do nothing" is also a decision, it means continuing on the current course.
- **The input is uncertain in kind, not just in degree.** Uncertain existence (is the storm real?), uncertain relevance (does it affect this airframe at all?), uncertain geometry (stationary or closing?), uncertain timing (is "12 minutes" a point estimate or a range?).
- **There is no oracle, before or after.** Not before: Earth is out of reach on this timescale (§1.2). Not after: if the aircraft reroutes and survives, it never learns whether the storm was actually lethal or the detour was wasted. Individual decisions can never be graded.

Stated abstractly: this is decision-making under uncertainty with no ground truth. The system must act on an unverifiable signal, and must be able to justify that action without ever checking it against an answer key.

## 1.2 Why the aircraft is on its own

The numbers in the prompt close a door, and it is worth closing it explicitly.

Storm contact is about 12 minutes ahead on the current route. The Earth round-trip is 14 minutes, roughly 7 each way. If the aircraft radios Earth the moment the storm is detected (T=0), the message reaches Earth at T+7, when the aircraft is 5 minutes from the storm. Even if Earth's analysis were instant and perfect, the reply arrives at T+14, **two minutes after the aircraft has already entered the storm.**

So "ask for help" is not a degraded option; it is not an option. Full decision authority for this class of event must live onboard. Autonomy here is not a design preference, it is arithmetic.

One corollary: the aircraft should still transmit its telemetry and intended action immediately. Not because the answer can help, it cannot, but because that record feeds the review loop on Earth (§2.6), which is the only mechanism by which the onboard decision logic ever improves.

## 1.3 The questions I would ask

I count a question as worth asking only if its answer would change what the system does. Each question below carries that test.

**A. The mission**

- *What is this aircraft trying to accomplish, and what does it cost to fail the mission versus to lose the vehicle?* → changes: everything. Without an objective, "what to do" has no defined answer, the same 60%-reliable detection means "turn back" for one mission and "push through" for another.

**B. The vehicle**

- *Will the aircraft even be affected by this storm, what are its structural and navigation dependencies on visibility?* → changes: whether this is a threat at all. If the airframe tolerates this dust density and navigates without visual reference, the right action may be "continue and log it."
- *What is the remaining energy margin, and can this airframe loiter, turn back, change altitude, land?* → changes: which escape options physically exist.

**C. The threat signal**

- *What is the detection source and how reliable is it, a single onboard sensor reading, or a fused forecast with an uncertainty band?* → changes: how much corroboration is needed before acting, and how far the decision threshold moves off 50/50.
- *Is "12 minutes ahead" distance-based (we approach a stationary storm), or is the storm moving toward us?* → changes: whether 12 minutes is a stable budget or a shrinking one, and whether "go around" is even geometrically possible.
- *Is the storm growing? What severity, at what confidence?* → changes: whether the hazard estimate has a usable shape or only a location.

**D. The environment**

- *Do we have a map of alternative routes and safe landing sites, and what is each one's time price?* → changes: which actions exist in the decision set at all (no mapped divert = no divert option), and how each is scored against the mission value curve rather than filtered by a deadline.
- *How stale is our knowledge of each alternative?* → changes: the confidence discount on mapped options, and whether the policy must verify on approach, treating a mapped site as a hypothesis to check with onboard sensors, not a fact.
- *What do we know about the divert corridor, from what source, at what freshness, compared to what we know about the storm?* → changes: whether diverting reduces risk at all. The storm is a characterized risk; an unsurveyed corridor is an uncharacterized one. Diverting is only rational when the corridor's uncertainty is better than the storm's.

**E. The system as built**

- *What is already installed on this aircraft, sensors, compute, the existing autopilot stack, and what is my modification budget: new hardware, or software only?* → changes: the shape of the whole solution. If the alarm-raising sensor is the only forward-looking one, there is nothing to corroborate against. If compute is modest, the decision logic must be cheap. If hardware can change, the highest-leverage proposal may be a second, independent hazard sensor: two independent detectors agreeing is a different class of signal than one detector being confident.

One question on this list produced a design insight before any design existed. Asking *"is the threat still there?"* implies the world keeps changing after the decision, so the decision cannot be one-shot. It must be re-evaluated continuously, and a divert can be un-diverted if the storm dissipates. That idea appears throughout Part 2.

## 1.4 Assumptions

The questions in §1.3 will not be answered before I must design, so I state my assumptions explicitly, and flag which ones are load-bearing: the ones that, if wrong, change the approach rather than just the numbers.

| # | Assumption | Why it's reasonable | Load-bearing? |
|---|---|---|---|
| 1 | **Mission:** the aircraft carries high-value, time-critical cargo between bases. Completing the delivery outranks preserving the vehicle, but cargo and vehicle are physically coupled, so losing the vehicle *is* losing the mission. | The prompt gives no mission, and without an objective "what to do" has no defined answer. This choice creates real tension (time pressure vs. safety) instead of a trivial always-avoid policy. | **Yes, the most load-bearing.** Patient cargo collapses the policy into "always avoid; time is free." A cargo-less scout decouples vehicle loss from mission failure and rewrites the whole cost model. |
| 2 | **Mission value decays smoothly with lateness**, a curve, not a hard deadline. | Real cargo value rarely disappears at one instant. And a hard deadline forces a binary decision at a line our estimates cannot resolve, two readings inside the same error band can command opposite actions. The curve keeps the decision consistent within our measurement error. | **Yes.** A true hard cutoff (e.g., an orbital rendezvous window) would need an explicitly different decision mode that accepts calculated gambles. |
| 3 | **Detection is probabilistic, not boolean**, a false-alarm rate, an ETA band, a severity distribution. | Any remote hazard detector on Mars is noisy; assuming near-perfect detection would be indefensible. | **Yes.** Near-certain detection would make most of the corroboration and confidence machinery unnecessary. |
| 4 | **Vehicle:** fixed-wing-class, can divert, climb/descend, turn back, land at suitable sites; cannot hover; loitering costs energy. | The prompt says "aircraft"; fixed-wing is the conservative capability assumption, a design that works with fewer escape options also works for more capable vehicles. | Partially. A rotorcraft adds "stop and wait" as a cheap option and softens the time pressure everywhere. |
| 5 | **Onboard compute is modest:** enough to probabilistically score a handful of candidate actions; not enough for heavy learned models in the decision loop. | Radiation-hardened space computers run generations behind consumer hardware. | No. More compute enriches the estimation but doesn't change the architecture. |
| 6 | **A route/terrain map with alternatives exists onboard, with known staleness.** | Flying between bases implies surveyed routes. Staleness is not assumed away, it becomes a confidence discount on every mapped option (§1.3-D). | Partially. With no map, the action set collapses to continue/climb/turn-back, and the design leans much harder on onboard sensing. |
| 7 | **An asynchronous telemetry link exists** (the 14-minute channel). | Given in the prompt. | No. It enables the improvement loop (§2.6), not the onboard decision. |

If one assumption deserves to be challenged first, it is #1: every downstream trade-off inherits its shape from what the mission is worth.

## 1.5 What makes this genuinely hard

I spend my working days operating machinery, so my instinct is to weigh difficulty from the operator's seat, what makes this hard for the thing that has to fly it. From that seat, ranked:

1. **The decision is one-shot and effectively irreversible.** There is no A/B-testing an entry into a dust storm. Most engineering intuition assumes iteration; this decision permits none. (One nuance from §1.3-D: *partial* reversibility exists, a divert can be un-diverted if the storm dissipates. The irreversibility is concentrated in exactly one action: continuing in. That is why reversible options get privileged status in the design, §2.1.)
2. **Uncertainty compounds.** Detection confidence × ETA band × severity estimate × airframe tolerance, each is merely uncertain; their product is a band wide enough that pretending to precision would be self-deception. Any policy that demands finer inputs than these is designed for an aircraft that doesn't exist.
3. **The objective is a value judgment, not a fact.** Physics will not say how much vehicle risk a delivery is worth; someone must set it. The coupling of cargo and vehicle (§1.3-A) settles the catastrophic end, losing the vehicle is losing the mission, there is no trade there, but the middle is live: every unit of caution buys risk reduction and costs mission value. Pricing that trade is the cost model of §2.3, and it is a choice the designer must own, not something derivable.
4. **The two error types cost differently.** A false alarm wastes limited energy and time; a miss can end everything. The decision threshold must sit off-center, and where it sits is, again, an owned choice.

But I am not in the seat this time. I am at the drawing board, designing the thing that will sit in it, and the drawing board carries two difficulties the operator never sees. They are the ones that make this problem genuinely novel:

5. **No outcome is ever graded.** An operator gets better because reality scores every maneuver. Here, a survived reroute never reveals whether the storm was actually lethal or the detour a waste. Experience accumulates flight hours, but not verdicts. This is what makes difficulty #1 permanent: a one-shot decision that could at least be graded afterward would still improve the fleet. One-shot *and* ungradeable is the real problem.
6. **The system must be trusted before it can be tested where it matters.** There is no rehearsal on real Mars. Whatever confidence we have in the decision logic must be built on Earth, indirectly, before the first storm is ever faced.

Terrestrial aviation faces difficulties 1-4 daily, and handles them with margins, checklists, and thousands of graded outcomes. What Mars removes is exactly the grading and the rehearsal. Difficulties 5 and 6 are why the standard medicine is unavailable, and they are what §1.6, §2.5 and §2.6 exist to answer.

Ranked honestly: from the cockpit, 1 > 2 > 3 > 4. From the drawing board, 5 and 6 dominate everything. This document is written at the drawing board, by someone whose instincts were formed in the seat; it tries to use both.

## 1.6 The hard question underneath: how would I ever know my decision logic is any good?

Sections 1.1-1.5 leave me in an uncomfortable position: I must ship a decision policy whose individual decisions can never be graded, not before flight (no rehearsal exists) and not after (no verdict ever returns). "Is this policy good?" has no direct test. So before designing anything, the question has to change shape: **if I cannot grade the answers, what evidence about the process that produces them would earn my trust?**

This situation is less exotic than it looks. Measurement systems get qualified without perfect reference standards all the time: you test whether the instrument agrees with itself, whether independent instruments agree with each other, and whether its stated uncertainty matches its actual scatter. None of that requires knowing the true value. The same logic applies to a decision system. Before betting a vehicle on this policy, I would demand four kinds of evidence:

1. **It must agree with itself.** Take one scenario, perturb the inputs within their stated error bands, and the chosen action must not flip. A decision that flips on noise is a decision about the noise, not about the world. (This demand already earned its keep once: it is why the hard-deadline model was rejected in §1.4 in favor of the decaying value curve.)
2. **Independent methods must agree.** Two estimators that share no assumptions, pointed at the same scenario, should reach compatible conclusions, and when they diverge, the divergence is itself a signal: confidence drops, and the policy should retreat toward reversible actions.
3. **Its confidence must be honest.** Across thousands of simulated scenarios, of the cases the policy labels "~80% safe," about 80% must actually end safely *within the simulation*. This proves nothing about Mars, but a policy that cannot keep its promises inside its own model fails the cheapest test we own.
4. **Its predictions must survive contact with telemetry.** The decision's verdict never comes back, but its *ingredients* do: ETA forecasts, energy consumption, and storm evolution are all eventually revealed by the data the aircraft sends home. Grade the ingredients, even though the final result can never be graded. Bad forecasts condemn a policy without any graded decision needed.

None of this proves the policy correct, that proof does not exist, which is the entire character of the problem. What the four demands do is convert "trust me" into an evidence trail, and, just as important, define **in advance** what losing that trust looks like, so "do not ship" and "fall back to the conservative default" are triggered by criteria, not by hindsight.

Stated once, generally: when a system's outputs cannot be checked against truth, you check the process that produces them, its consistency, its internal agreement, its honesty about uncertainty, and the accuracy of every intermediate claim that reality does let you verify. Section 2.5 turns each of these demands into a concrete test harness.

---

# Part 2, Proposed Solution

## 2.1 Design principles

Four principles, each forced by a specific finding in Part 1, none of them free-floating best practices.

**P1: A hard safety floor that no score can override.** No action whose estimated probability of vehicle loss exceeds a fixed bound is ever selectable, regardless of how much mission value it promises. ← Forced by the coupling in §1.3-A: cargo and vehicle are tied together, so no amount of mission value can compensate for losing the vehicle, the trade the optimizer would be making does not actually exist. Structurally, this removes a whole class of gambles from the optimizer's reach: it cannot be tempted by options it cannot see.

**P2: Under low confidence, prefer actions that preserve options.** When evidence is thin or independent estimators disagree (§1.6-2), choose the action that keeps the most future actions available, even at some mission-value cost. ← Forced by §1.5-1: the irreversibility is concentrated in one action (continuing in), and by the re-evaluation insight of §1.3, a preserved option can still be exercised when the picture clears; a spent one cannot.

**P3: The final decision must be inspectable.** Every selected action ships with its inputs, its scores, and the rule that chose it, in a form a human on Earth can read. ← Forced by §1.5-5/6: the slow loop (§2.6) is the *only* mechanism by which this policy ever improves, and an unauditable decision gives that loop nothing to work with. This is an owned trade-off: it argues against a black-box learned policy at the final decision stage, and I accept the possible performance cost, in a no-ground-truth setting, an unauditable policy that fails leaves behind nothing to learn from, which makes its failure permanent in exactly the way §1.5-5 describes.

**P4: Degrade toward a defined safe default.** Sensor conflict, estimator divergence, or a collapse in confidence must land the aircraft somewhere *designed*, not somewhere accidental. ← Forced by §1.5-2: compounding uncertainty guarantees that "I don't know" states will occur; a policy without a designed answer to them has an undesigned one. The default itself is specified in §2.3.

## 2.2 Architecture: the decision loop

```
        ┌─────────────────────────────────────────────────────────┐
        │                    ONBOARD (fast loop)                    │
        │                                                           │
  sensors ─▶ [1] PERCEPTION ─▶ [2] RISK/OPTIONS ─▶ [3] DECISION ─▶ [4] ACT
        │      (belief about      (feasible actions   POLICY          │
        │       world + self)      + expected cost)   (choose safe)   │
        │            │                                    │           │
        │            └──────────▶ [5] LOG / TELEMETRY ◀───┘           │
        └───────────────────────────────│──────────────────────────-─┘
                                         │ (async, ~7 min each way)
                                         ▼
                           EARTH (slow loop, see §2.6):
                           offline review, tune policy, re-upload
```

**[1] Perception / state estimation.** Fuses sensor data into a probabilistic world model: `P(hazard is real)`, an ETA band, a severity distribution, plus vehicle state (energy, position, health). The storm is never a boolean, §1.1 established the uncertainty is in kind (existence, relevance, geometry, timing), and a yes/no flag throws all that structure away. A flag also makes the §1.6 demands untestable: you cannot check the calibration of a yes/no.

**[2] Risk / options assessment.** Builds the feasible action set, and "feasible" is earned, not assumed (§1.3-D): an action exists only if backed by current world knowledge. Each option carries its **time price** (scored against the value curve, never filtered by a deadline), its **staleness discount** (mapped assets are hypotheses, not facts), and its **risk characterization** (a surveyed corridor and an unsurveyed one are different objects, even at equal estimated risk).

**[3] Decision policy.** Chooses the action, full logic in §2.3. Deliberately the simplest block: scores in, rule applied, action out, everything inspectable (P3).

**[4] Actuation + monitoring.** Executes, and keeps deciding. The world model keeps moving after commitment (§1.3-D), so the loop re-evaluates continuously: a divert is un-diverted if the storm dissipates; a "continue" is abandoned if the signal strengthens. Execution also watches **predicted-vs-actual** in real time, drift between them is an early warning that the models feeding decisions are wrong (§1.6-4, applied onboard).

**[5] Logging / telemetry.** Every decision ships home immediately with its inputs, scores, chosen rule, and confidence (§1.2: not because Earth can help, it can't, but because this record is the raw material for §2.5's validation and §2.6's improvement loop). What cannot be graded can at least be audited.

## 2.3 Decision logic

```python
def decide(world, vehicle, mission, limits):
    # world:   probabilistic belief, P(hazard), ETA band, severity dist, map + staleness
    # vehicle: position, energy, health, airframe tolerance envelope
    # mission: value curve V(lateness), decaying, not a cliff (§1.4 row 2)

    actions = feasible_actions(world, vehicle)     # feasibility is earned, not assumed (§2.2[2])

    scored = []
    for a in actions:
        p_loss = upper_bound_loss_prob(a, world, vehicle)  # conservative percentile, not the
                                                           # mean, the errors are asymmetric (§1.5-4)
        cost   = expected_value_lost(a, world, mission)    # the action's time price, priced on the
                                                           # value curve; energy spent checked vs reserve
        conf   = confidence(a, world)                      # independent-estimator agreement ×
                                                           # input freshness (§1.6-2, §1.3-D)
        scored.append((a, p_loss, cost, conf))

    # 1) SAFETY FLOOR, a constraint, not a cost term (P1).
    #    Vehicle loss cannot be priced (coupling, §1.3-A), so it is not traded, it is excluded.
    safe = [s for s in scored if s.p_loss <= limits.max_loss_prob]
    if not safe:
        return safe_default(world, vehicle, limits)

    # 2) CONFIDENCE GATE (P2), below threshold, do not optimize; preserve options.
    best = min(safe, key=lambda s: s.cost)
    if best.conf < limits.min_confidence:          # threshold tuned in simulation via the
        return most_option_preserving(safe)        # calibration harness (§2.5); held with
                                                   # hysteresis so it cannot flip on noise (§2.4)
    # 3) Otherwise: the cheapest safe action on the value curve.
    return best.action
    # Every exit path emits: inputs, scores, rule fired, confidence -> log (P3, block [5])


def safe_default(world, vehicle, limits):
    # Loiter is a state you pass through, not a destination.
    hold = hold_point_outside_projected_threat(world)      # the storm may be moving (§1.3-C):
                                                           # holding still in its path is not holding
    if vehicle.energy > limits.reserve_floor and hold is not None:
        return Loiter(at=hold, until=first_of(
            confidence_recovers(),      # picture cleared -> resume normal scoring
            timeout(),                  # clarity never came -> escalate
            energy_floor()))            # can no longer afford to wait -> escalate
    # Escalation: retreat to the BEST-CHARACTERIZED option (§1.3-D) ,
    # turn back along the just-flown route (freshest knowledge the vehicle owns)
    # vs. land at a mapped site (a stale-map hypothesis), freshest wins.
    return best_characterized(TurnBack, LandAtNearestSite)
```

**In plain words, what this code does.** For every move the aircraft could make right now (continue, divert, climb, loiter, turn back, land), it computes three numbers: how *dangerous* the move is (the probability of losing the vehicle, estimated pessimistically), how *expensive* it is (how much mission value its delay burns, using the value curve), and how *confident* the system is in the inputs behind those two numbers (do the independent estimators agree, and how fresh is the data). Then three rules apply in order. First, any action above the danger limit is deleted, a dangerous action is not "expensive," it is unavailable. Second, if the best remaining action was scored on shaky inputs, the system does not take it; it takes the action that keeps the most options open instead, which is usually loiter. Third, if the inputs are trustworthy, it simply takes the cheapest safe action. Whatever happens, the full record, measurements, scores, which rule fired, goes to the log.

The `safe_default` function is the "when in doubt" behavior: hold at a point *outside the storm's projected path*, if energy allows, and wait for the picture to clear. Loiter has three exits, confidence recovers, a timeout expires, or energy hits its floor, so waiting can never quietly turn into a fuel-starved forced landing. If loiter is not affordable, the aircraft retreats to whichever backup option it knows the most about: usually turning back along the route it just flew, because that is the freshest information it owns.

**The cost model.** Mission value is a function `V(lateness)` that decays smoothly from full (on-time) toward zero, the curve of §1.4 row 2. Every action is charged its time price against this curve; nothing is ever filtered by a deadline. Vehicle loss is deliberately **not** a term in this model: because cargo and vehicle are coupled (§1.3-A), losing the vehicle zeroes all remaining value, so it enters as a constraint (the P1 floor) rather than a cost, a trade that must never be made should not be computable. The asymmetry of §1.5-4 appears as a modeling choice: loss probabilities use conservative upper percentiles, time costs use expected values. The system is deliberately harder to convince about safety than about schedule.

**Where learning is allowed, and where it is banned.** Learned or statistical estimators are welcome in block [1], the perception layer: hazard probability, ETA bands, severity, exactly the quantities that telemetry eventually reveals, so §1.6-4 can grade them against reality. The final selection in block [3] stays a fixed, inspectable rule. The dividing line is not taste; it is gradability: **learning is allowed exactly where reality can grade it, and rules are required exactly where nothing can.** A learned component that drifts in the perception layer gets caught by predicted-vs-observed monitoring. A learned final policy that drifts would fail silently, with no verdict ever coming back to expose it.

## 2.4 Data flow: from raw signal to a trusted belief

The pipeline: raw sensor readings → per-sensor filtering → fusion into the probabilistic world model of §2.2[1], with uncertainty carried forward at every stage, never collapsed early. A reading is never a fact; it is a fact plus an error bar plus an age, and all three travel together into the decision.

**Noise rejection: persistence, symmetric by design.** The belief does not flip on any single reading, in either direction. A transition toward "hazard is real" and a transition back toward "clear" both require the same sustained corroboration across consecutive updates. This symmetry is a deliberate decision, and the reasoning matters: the caution this system needs is **already encoded once, explicitly, in the decision layer**, conservative percentiles on loss estimates, the P1 safety floor, the confidence gate. Biasing the perception layer as well would apply caution twice, and the two biases would multiply into a total safety margin that no one can state or tune. The principle: **perception reports the world as honestly as it can; all of the conservatism lives in the decision layer, on the record.** An instrument deliberately made pessimistic "for safety," sitting behind an already-conservative acceptance rule, does not make a system safer, it makes its margin unknowable. The economics agree: a false fear held longer than the evidence warrants is pure mission-value loss that bought nothing.

**Disagreement is information.** Where the hardware budget allows, hazard estimation runs on two independent paths, ideally two sensing modalities (§1.3-E), at minimum two algorithmically independent estimators on the same raw feed. Their agreement feeds the confidence term in §2.3 directly: convergence raises it; divergence drops it, which pulls the policy toward the option-preserving branch and, if severe, into the loiter ladder. The system never averages a disagreement away, two witnesses telling different stories is a finding, not a rounding problem.

**Freshness as a first-class input.** Every input carries its age, and age decays confidence, the staleness discount of §1.3-D, generalized from map data to all data. A belief built on old readings may have the same nominal value as one built on fresh readings; it does not carry the same weight, and the confidence term knows the difference.

## 2.5 Validation: how I earn confidence in a system I can't field-test

Section 1.6 established what evidence would earn trust in a policy whose decisions are never graded. This section turns each of the four demands into a harness that produces that evidence, plus the follow-up every such plan owes its reader: what grades the grader?

**The scenario engine.** Everything below runs on a simulator whose core job is not realism but *coverage*: a scenario generator that samples across the full uncertainty space of Part 1, storm severity, position and drift; ETA error; sensor noise and false-alarm behavior; vehicle energy states; map staleness; value-curve steepness. Thousands of sampled scenarios per policy version, so every claim below is a statement about a distribution, never about a handful of hand-picked cases. Outcome metrics per run: vehicle survival, mission value delivered (on the curve), energy consumed, time in loiter, escalations triggered.

**Harness 1: Stability** *(demand: it must agree with itself)*. Take a scenario, hold the world fixed, jitter every input within its stated error band, re-run the decision. The chosen action must be stable within the band; the flip rate near decision boundaries is measured and budgeted. This is a direct, automated test of the property that killed the hard deadline in §1.4, now enforced continuously instead of argued once.

**Harness 2: Cross-agreement** *(demand: independent methods must agree)*. The two estimation paths of §2.4 run on every simulated scenario, and their divergence is scored: how often, how large, and, critically, whether divergence *predicts* bad outcomes in simulation. If it does, the confidence gate is earning its keep. If divergence and outcomes turn out to be unrelated, the confidence term is decoration, and it must be rebuilt.

**Harness 3: Calibration** *(demand: its confidence must be honest)*. Bin the policy's stated safety estimates across thousands of runs and compare each bin against realized outcomes in simulation: of everything labeled "~80% safe," did about 80% end safely? The output is a reliability curve, and its drift across policy versions is tracked like a vital sign. Calibration inside the simulator proves nothing about Mars, but a policy that cannot keep its promises inside its own model has failed the cheapest test available, and has no business proceeding to the expensive ones.

**Harness 4: Prediction audit** *(demand: predictions must survive telemetry)*. Every real flight's log (§2.2[5]) contains the intermediate predictions each decision relied on: ETA forecasts, energy-burn projections, storm evolution estimates. Reality eventually reveals most of these, not whether the *decision* was right, but whether its *ingredients* were. Each mission's telemetry is scored against its own predictions, and the residuals are tracked over time. Growing residuals mean the models feeding the decisions are drifting away from the real Mars, detectable without a single graded decision.

**What grades the grader?** The simulator is itself a check that can be wrong, and it gets the same treatment it gives the policy. Its predictions are compared against every piece of real telemetry Harness 4 collects; the sim-to-real residual is quantified and carried as additional uncertainty on every simulator-derived claim (a policy "95% safe in sim" with a 10-point sim-to-real gap is treated as what it is, not what we wish it were). The recursion is not infinite: each layer is graded by the layer of reality beneath it, and the bottom layer, telemetry, is the one thing in this system that is never a model.

**Defined distrust.** Trust that cannot be lost is not trust. The red flags are declared in advance, with responses attached: calibration drift beyond budget → freeze policy updates; rising estimator-divergence rate → widen the confidence gate (the aircraft loiters more and delivers later, and that is the correct trade); growing prediction residuals → revert to the conservative baseline policy until the gap is explained. "Do not ship" and "fall back" are triggered by criteria written today, not by hindsight after a lost vehicle.

Stated once, in general form: this is the standard shape of validating any automated decision system that has no labels. Individual outputs cannot be graded, so the process is graded instead, its stability under noise, the agreement of independent methods, the honesty of its stated confidence, and the gap between what it predicted and what reality later revealed. The aircraft is one instance of that problem; the method is not specific to aircraft.

## 2.6 The role of Earth (the slow loop)

The arithmetic of §1.2 removes Earth from the decision; it does not remove Earth from the system. It changes *when* Earth's judgment enters, and the design splits that judgment cleanly around the flight.

**Before flight, Earth decides the things that are choices, not facts.** Section 1.5-3 established that the objective is a value judgment physics cannot supply: what the cargo is worth, how steeply its value decays, how much loss probability is ever acceptable, how much confidence is enough. These are exactly the numbers in `mission` and `limits` in §2.3, and they are set by humans, on Earth, with time to argue about them. The aircraft executes value judgments; it never invents them.

**After flight, Earth is the improvement loop, the only one that exists.** Every decision comes home as an auditable record (P3: inputs, scores, rule fired, confidence). Earth's job is the part of §2.5 that needs human judgment: reviewing decisions the harnesses flag, investigating prediction residuals, retuning weights and thresholds, and uploading the revised parameters for the next flight. Because the policy is parameterized rules rather than opaque code (P3 again), an upload is a reviewable diff of numbers, not a rewrite of logic that could smuggle in new, untested behavior.

The division of labor in one line: **the aircraft owns the twelve minutes; Earth owns everything before and after them.** Any design that puts a human inside the twelve minutes is denying arithmetic; any design that keeps humans out entirely has cut the only path by which the policy ever gets better. Human *on* the loop, auditing and steering it, rather than *in* it.

## 2.7 Failure modes → how the design answers them

Part 1 opened six difficulties; this table shows where each one is handled, and, honestly, what risk remains after the mechanism has done its work. No row claims zero residual, because no mechanism in this document eliminates uncertainty; they price it, gate it, or contain it.

| §1.5 difficulty | Mechanism that answers it | Residual risk that remains |
|---|---|---|
| **1: One-shot, irreversible decision** | P2 + confidence gate: under uncertainty the policy buys reversible options (loiter ladder, §2.3); irreversibility is concentrated in one action ("continue in"), which sits behind the P1 safety floor | A scenario can close all reversible exits (low energy + no hold point) and force a committed choice on thin evidence, the escalation ladder picks the best-characterized retreat, but "best" may still be poor |
| **2: Compounding uncertainty** | Uncertainty carried as distributions end-to-end (§2.4); decisions scored on bands, not points; Harness 1 verifies decisions don't flip inside stated error bands | The *stated* bands may be wrong, overconfident inputs poison every downstream computation; caught only slowly, by Harness 4 residuals |
| **3: Objective is a value judgment** | Value curve + limits set by humans on Earth, before flight (§2.6); the aircraft executes judgments, never invents them | The judgment can simply be wrong (cargo valued incorrectly, curve too steep or too shallow), the system will faithfully optimize a mistaken objective; only the slow loop can notice and revise it |
| **4: Asymmetric error costs** | Asymmetry encoded once, explicitly, in the decision layer: conservative percentiles on loss, expected values on time (§2.3); perception stays honest (§2.4) | The chosen percentile is itself a judgment, too conservative bleeds missions on false alarms; too bold erodes the floor's meaning; tuned only in simulation (§2.5), which has its own gap |
| **5: No outcome is ever graded** | Grade the process instead: stability, cross-agreement, calibration, prediction audits (§2.5, Harnesses 1-4); auditable decisions (P3), so ungradeable ≠ unexaminable | The proxies can all pass while the policy is still wrong in a way only real outcomes would reveal, this residual is irreducible, and it is the honest core of the problem |
| **6: Must trust before real-world test** | Scenario-engine coverage across the uncertainty space; the simulator graded against telemetry (§2.5); distrust criteria defined in advance, with pre-committed fallbacks | The sim-to-real gap is estimated from sparse telemetry, early missions fly on the least-validated version of the system; the conservative baseline policy is the only real protection there |

One deliberate observation: the two residuals that cannot be engineered away (rows 5 and 6) are the same two difficulties ranked highest from the drawing board in §1.5. The design does not pretend to close them, it surrounds them with instrumentation so that when they bite, the bite is visible, bounded, and survivable. That is the honest limit of what engineering can do against a problem with no answer key.

## 2.8 Tools & technologies

A rule for this section: the design constrains *properties*, and names a specific tool only where a specific reason exists. Where none does, "any X that does Y" is the honest answer.

- **Scenario engine:** a custom, lightweight simulator rather than an off-the-shelf physics or game engine. The job is coverage of the uncertainty space, many thousands of cheap, statistically varied runs, not visual or aerodynamic fidelity. A Python/NumPy-class stack fits: fast to iterate, strong statistical ecosystem, and the whole §2.5 analysis pipeline lives next door to it.
- **State estimation / probabilistic modeling:** standard, well-understood filtering and sampling techniques (Kalman-family or particle filters for state; Monte Carlo sampling for outcome distributions). Any competent implementation is acceptable, the technique matters, the brand does not.
- **Onboard decision logic:** plain, deterministic code implementing the §2.3 rules, parameterized thresholds, no frameworks, no runtime dependencies beyond the flight stack. This is forced twice over: by modest onboard compute (§1.4 row 5) and by P3's inspectability demand. Implementation language deferred to whatever the existing avionics stack uses (§1.3-E: this is a retrofit, not a greenfield).
- **Logging & telemetry:** append-only, structured, schema shared between the aircraft and Earth, the same record drives Harness 4, the calibration tracking, and human review, so there is exactly one version of what happened.
- **Earth-side analysis:** ordinary data tooling, dataframes/SQL for the log store, notebooks for investigation, standing dashboards for the §2.5 vital signs (reliability curves, divergence rates, prediction residuals). Nothing exotic; the value is in what is measured, not in the dashboard brand.

## 2.9 What I'd build first

The build order falls out of the drawing-board conclusion of §1.5: the scarce capability here is not *making* decisions, it is **knowing whether a decision-maker can be trusted**. So the first thing built is the machine that measures trust, and the first policy through it is deliberately simple.

1. **The scenario engine and outcome metrics** (§2.5). Without it, every threshold, percentile, and curve parameter in this document is a guess.
2. **Harnesses 1-3, run against a trivial baseline policy**, something as blunt as *"any credible hazard → loiter; timeout → turn back."* This produces the baseline numbers: survival rate, mission value delivered, calibration honesty. The baseline policy is not a placeholder to be thrown away, it is the **permanent fallback** that §2.5's distrust criteria retreat to. Building it first means the safety net exists before anyone walks the wire.
3. **The §2.3 scoring policy**, admitted into the fleet only when it beats the baseline *inside the harnesses*, on the same scenario distribution, at equal or better calibration. The sophisticated policy earns its seat; it is never granted it.
4. **The telemetry schema and Harness 4, live from the first real flight.** The prediction audit cannot be retrofitted, every flight that happens before it exists is validation data lost forever.

An engineer joining tomorrow starts at step 1, and nothing in steps 2-4 blocks on aerospace expertise: the simulator, the harnesses, and the baseline are ordinary software with unusual consequences.

---

## Closing note

The dust storm appears in this document's title and almost nowhere in its solution. That is deliberate. The storm was the occasion, not the problem: the problem was making a decision worth trusting, alone, in minutes, from signals that may be wrong, and being able to keep trusting the system that makes such decisions, when no answer key exists before, during, or after. The design above is my honest attempt at that problem. Its center of gravity is the validation layer, where the system either earns its trust with evidence or loses it against criteria written in advance, which is exactly where the center of gravity belongs.

---

## Appendix: How AI was used in this work

I used an AI assistant (Claude) throughout this challenge, and I want to be transparent about exactly how, because the division of labor matters.

**The ground rules I set.** At the start I agreed on a working mode with the assistant: it scaffolds and pressure-tests, I make and own the decisions. The assistant was explicitly not asked to "solve the challenge", it was asked to structure my thinking, argue against my positions, teach me concepts I was missing, and draft prose from positions I had already taken and defended. Later, when the drafting was done, I asked it to cut over-polished phrasing so the document reads in my voice, and to add a plain-language explanation of the pseudocode, a document whose author cannot explain its own code is not the author's document.

**What was mine.** The exploration questions that open §1.3 (detector reliability, storm geometry, whether the storm even affects this airframe, the map's time prices, the corridor's uncharacterized risk, the as-built system); the mission assumption and the time-critical cargo call; the choice of a decaying value curve over a hard deadline; the difficulty ranking in §1.5 and its operator's-seat framing; loiter as the safe default; and the symmetric-persistence decision in §2.4.

**What was the assistant's.** The document structure; the pressure-testing that exposed weaknesses in my first positions (for example, walking me through what a hard "late = worthless" deadline does to the decision math, after which I changed my position, and the two-part rationale in §1.4 row 2 is the result); teaching concepts like calibration and hysteresis through analogies to measurement-systems practice from my quality background; the formalization of my decided logic into the §2.3 pseudocode; and drafting the prose of this document from the decisions above, which I then reviewed.

**Where we disagreed, and how it resolved.** Two examples, because they show the management more honestly than any claim could:

1. *Belief-flipping symmetry (§2.4).* The assistant proposed the classic safety pattern: quick to believe "hazard," slow to believe "clear." I pushed back, every minute of unwarranted fear costs mission value, and the system's caution already lives in the decision layer. The final design is symmetric, and the "one bias, one place" reasoning in §2.4 came out of that argument. The assistant conceded the point.
2. *Difficulty ranking (§1.5).* I ranked irreversibility first, because I operate machinery for a living and instinctively judge from the seat. The assistant argued the no-ground-truth difficulties should dominate, because this is a design document. Neither position fully won: §1.5 keeps my ranking from the cockpit and adopts the designer's tier for everything that follows. The two-lens structure of that section *is* the record of the disagreement.

**Representative prompts.** My side of the conversation was mostly short and decisional rather than generative, a fair sample: *"What's the detection source and its reliability? Is '12 minutes ahead' distance-based, or is the storm moving toward the aircraft? Will the aircraft actually be affected by this dust storm?"* (my opening questions, before any design existed); *"We're going with the curve, the cliff makes it too much of a gamble we cannot afford"*; *"Hold/loiter"*; *"Fear equal to relax time, every minute counts"*; *"I'll defend my ranking: I'm usually the one maneuvering machinery, and it comes naturally to put myself in that space."*

Full conversation logs are available on request.


