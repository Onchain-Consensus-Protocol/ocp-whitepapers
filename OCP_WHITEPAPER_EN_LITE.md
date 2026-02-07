# OCP — On-Chain Consensus Protocol (Lite)

*A short overview for forum discussion. Full whitepaper and mechanism details are available separately.*

---

## One-liner

**If Bitcoin gave us value on-chain, OCP aims to give us facts on-chain**—a permissionless layer to turn real-world disputes into programmable, finalized on-chain consensus, without oracles or a single authority.

---

## The gap we’re targeting

Real-world questions (“who won?”, “did it happen?”) are hard to get as **trustworthy, programmable finality** on-chain. Oracles add a single point of failure; on-chain courts often decouple consensus from capital; prediction markets still lean on external settlement. The “last mile” of consensus—bringing a dispute on-chain and resolving it irreversibly—is still messy.

---

## Core idea

- **Consensus as a settleable asset.** OCP is a thin protocol layer: anyone can bring a binary dispute on-chain; the protocol finalizes it through a **stake–challenge** game.
- **Proof of Commitment (POC).** Authority comes from **verifiable capital at risk**, not identity or claims. We don’t assume people tell the truth; we assume being wrong is expensive.
- **Three outcomes:** YES, NO, or **INVALID** (event cancelled / unverifiable / ambiguous). INVALID works as a circuit breaker and refunds everyone instead of forcing a binary result.

Output is **Finalized Consensus**—one canonical on-chain fact per event that contracts and other apps can read. We frame it as “capital-confidence”: what outcome capital is willing to lock in, not a claim about objective truth.

---

## Why it might matter for Ethereum

- **Oracle-free prediction markets** and other apps that need “fact finality” without a central data source.
- **Optimistic-style governance**: propose by staking; default pass unless someone stakes to challenge.
- **Sync / L2 context**: as execution gets faster, “is the input true?” remains slow. OCP is designed so consensus can be referenced at “same-speed” for atomic flows (economic pre-finality, INVALID as a circuit breaker when facts are ambiguous).
- **Longer-term**: a programmable layer for “value on-chain”—not only facts but preferences and alignment targets—readable by contracts and agents.

---

## What we’re not claiming

We’re not claiming OCP outputs “truth.” It outputs **consensus that was expensive to form and to challenge**—which is what many applications need. It works best for **publicly verifiable, time-bounded** events; edge cases (non-verifiable info, heavy ideology, no one willing to challenge) can produce “expensive errors.” That’s the intended design boundary.

---

## Where to go from here

- Full narrative and design goals: **OCP Whitepaper**
- Mechanism, game theory, bond rules: **POC Whitepaper**
- First application: **Oracle-free prediction market** on top of OCP

Happy to discuss design tradeoffs, L2/composability use cases, or integration ideas in the thread.
