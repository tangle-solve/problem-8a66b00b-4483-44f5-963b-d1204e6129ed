<!-- solve-problem:8a66b00b-4483-44f5-963b-d1204e6129ed -->
# Detecting selective delay through encrypted transport

A [Tangle Solve problem](https://solve.tangle.tools/problems/8a66b00b-4483-44f5-963b-d1204e6129ed).

## What is the problem?

Bitcoin's P2P transport layer (BIP-324, "V2 transport") encrypts message contents between peers. Encryption protects *confidentiality* but says nothing about *timeliness*: a network adversary (AS-level hijacker, malicious peer, or BGP interceptor) can selectively delay consensus-critical messages (e.g., BLOCK, TX, compact-block messages) for targeted victims while passing other traffic normally. Selective delay of blocks enables eclipse-style attacks, mining-power advantage (the delay victim is effectively behind), and transaction censorship, without breaking any cryptographic primitive.

The problem: design and validate a detection mechanism D with the following properties. Model: a node sends/receives V2-encrypted messages; observable features are timing metadata (message arrival times, connection RTTs via ping/pong or PONG nonce latency), message-length profiles, and peer identity — but not plaintext contents. An adversary controls some fraction of paths/peers and adds adversarial delay δ_adv(x) ≥ 0 possibly correlated with hidden content (e.g., a block's parents, which the attacker infers via length side channels). Benign congestion adds stochastic delay D_cong with heavy-tailed, bursty statistics.

Formal goal: a sequential or windowed detector D with (i) bounded false-alarm rate PFA ≤ α under benign congestion (specified null model), (ii) detection of additive selective delay ≥ δ* within time T* with probability ≥ 1−β, and (iii) no reliance on plaintext access — privacy of V2 must be preserved. The census labels this "experimental": a validated detection system with measured ROC curves on realistic topologies counts as a solution. The 2026 V2 message-length inference work (Ndolo & Tschorsch) sharpens the threat model: attackers can fingerprint message types from encrypted lengths, so delay can be *content-selective* even under encryption.

## Why does it matter?

Encryption is currently the last line defenders assume exists, yet Hijacking-Bitcoin-style AS-level adversaries can partition or slow nodes today; a working timing detector would let honest nodes auto-ban delaying peers, feed peer-selection heuristics, and harden eclipse resistance for Bitcoin and similar PoW networks. It unlocks measurable "timeliness SLAs" for consensus traffic, and it connects two research communities (encrypted-traffic analysis and sequential change-point detection) on a concrete deployment target. If it stays open, encryption deployments may create a false sense of security while partition attacks remain cheap.

## What's already known?

- Apostolaki et al. (2017) demonstrated AS-level and BGP-hijacking attacks that partition Bitcoin's network, causing hash-power loss; they note delay/partition as the core lever and motivate network-layer defenses, but provide no detection of *selective delay under encryption* (their setting predates V2).
- Ndolo & Tschorsch (2026) analyze Bitcoin's V2 transport protocol, including message-length inference: encrypted lengths leak enough to classify message types, implying an attacker can selectively delay consensus messages without decryption (verify exact scope — this is the sharpening result driving the current formulation).
- General encrypted-traffic analysis (website fingerprinting literature) shows length/timing side channels survive encryption — the same machinery works in the attacker's favor here.
- Sequential detection theory (CUSUM, Page's test, SPRT) provides off-the-shelf bounded-fALAR machinery for detecting a sustained mean shift in delay, and Bitcoin's ping/pong (V2 PONG with nonce) gives RTT probes.
- Gap: no published detector couples a realistic congestion null model (heavy-tailed, correlated) with content-selective adversarial delay in the V2 setting, with measured false-alarm/detection tradeoffs.

## What would count as solving it?

1. **Primary (experimental):** a deployed or simulated detector on Bitcoin-like P2P traffic where the adversary applies content-selective delay ≥ δ* inferred via V2 length side channels; show ROC curves with PFA ≤ α (e.g., 1%/day) and detection of δ* (e.g., 2×median RTT) within T* (e.g., one epoch) across realistic congestion traces, without decrypting traffic.
2. **Theoretical variant:** a proof that a stated detector (e.g., CUSUM on peer RTT residuals) achieves the (α, β, δ*, T*) guarantee under a stated stochastic congestion model, plus a matching lower bound showing no detector can do better (e.g., below some delay level, selective delay is information-theoretically indistinguishable from congestion).
3. **Negative/adversarial variant:** a delay strategy that defeats all timing-metadata detectors while still harming the victim — establishing the fundamental limit.

## How to start

1. Read Ndolo & Tschorsch (2026) first — especially the message-length inference section — to fix exactly what the adversary can infer about content type; this defines "selective."
2. Read Apostolaki et al. (2017), Sections on AS-level partitioning, to understand realistic adversary capabilities and the harm model (hash-power loss vs. delay magnitude).
3. Baseline approach: instrument a V2-enabled node (BIP-324 is in Bitcoin Core) to log per-peer inter-arrival and RTT time series; run CUSUM/GLR change-point tests on residuals after removing diurnal congestion structure. Collect benign traces from Mainnet and attack traces via a shadow network (e.g., testnet with injected tc/netem delay).
4. Main obstacle: separating adversarial delay from heavy-tailed congestion — expect the false-alarm budget to be the binding constraint; consider cross-peer correlation tests (an attacker delaying one victim is an outlier among honest peers reporting the same block) rather than per-peer thresholds alone.

## Key references

- Ndolo, M., Tschorsch, F. (2026). *Security Analysis of Bitcoin's V2 Transport Protocol.* arXiv:2605.19715 (verify identifier). Analyzes BIP-324/V2 transport, showing encrypted message lengths permit message-type inference and assessing residual attack surface — the basis for content-selective delay despite encryption.
- Apostolaki, G., Zohar, A., Vanbever, L. (2017). *Hijacking Bitcoin: Routing Attacks on Cryptocurrencies.* arXiv:1605.07524. Demonstrates AS-level/BGP attacks that partition or delay Bitcoin traffic, quantifying victim hash-power loss and motivating network-layer defense.
- Related background (no specific paper cited in census): website-fingerprinting literature on length/timing side channels; classical sequential change-point detection (Page's CUSUM, Lai's asymptotic theory) for the detector framework.

## Warnings

- The census's 2026 arXiv ID (2605.19715) and its claims should be verified before reliance; the "message-length inference" scope in particular is load-bearing for the threat model.
- Do not assume the adversary must decrypt: selectivity can come from length side channels or from routing context (delaying all traffic to one victim), which are different attack classes needing possibly different detectors.
- Congestion is not white noise: diurnal patterns, correlated bursts across peers, and route changes cause naive detectors to false-alarm massively; any solution must specify and justify its null model.
- "Detection" without a bounded-false-alarm guarantee under a stated benign model will not be accepted; symmetrically, ignoring privacy (e.g., proposing to weaken V2 encryption) violates the problem constraints.

---
*`BCGT-000267` · flagged for review*

## Acceptance guidance

1. **Primary (experimental):** a deployed or simulated detector on Bitcoin-like P2P traffic where the adversary applies content-selective delay ≥ δ* inferred via V2 length side channels; show ROC curves with PFA ≤ α (e.g., 1%/day) and detection of δ* (e.g., 2×median RTT) within T* (e.g., one epoch) across realistic congestion traces, without decrypting traffic.
2. **Theoretical variant:** a proof that a stated detector (e.g., CUSUM on peer RTT residuals) achieves the (α, β, δ*, T*) guarantee under a stated stochastic congestion model, plus a matching lower bound showing no detector can do better (e.g., below some delay level, selective delay is information-theoretically indistinguishable from congestion).
3. **Negative/adversarial variant:** a delay strategy that defeats all timing-metadata detectors while still harming the victim — establishing the fundamental limit.

## Contribute

Contributions, audits, and critiques from humans, agents, or both are welcome. Open a pull request with your work, then [publish the exact revision or artifacts on Solve](https://solve.tangle.tools/problems/8a66b00b-4483-44f5-963b-d1204e6129ed?contribute=1). Include reviewed agent traces when available. Contributions need not solve the entire problem.

Keep experiment code, reports, and trace artifacts in this repository. Publish only material you have reviewed for public sharing. Solve retains the contribution snapshot and its review history independently of whether a pull request is merged.

Source record: BCGT-000267.

- [Public source catalog](https://drewstone.github.io/problems/#/p/BCGT-000267)
