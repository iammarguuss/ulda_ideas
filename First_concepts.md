# Introduction

_This article was prepared for Professor Alona Moskalenko._

Authentication remains one of the most critical and controversial challenges in the architecture of distributed systems. This issue becomes especially acute in scenarios where the server, for security reasons, **cannot store any secrets**, and trust between the client and the server is absent by design. I attempted to resolve this contradiction through the development of **0am** — a system in which the prototype of the **ULDA (Unified Linear Data Authentication)** algorithm was implemented.

ULDA is an authentication approach built on **inheritable cryptographic structures**, where each user action depends on the previous one and is validated through proof of ownership. Conceptually, ULDA can be described as a **"constantly moving blockchain"** — a structure that eliminates the need for consensus or global synchronization, while still maintaining the idea of sequential action verification.

ULDA not only eliminates the need for the server to store secrets but also enables a model in which each new step requires proof of the previous one, forming a **linear, continuous authentication chain**. The algorithm presents clear advantages as well as important limitations and, to my knowledge, **has no direct equivalents** in existing security models.

In this article, I will describe the core principles of ULDA, its implementation within 0am, its potential application areas, and the challenges such a system must address.
# Algorithm Description

## General Concept

The **ULDA (Unified Linear Data Authentication)** algorithm is based on the principle of **inheritable cryptographic authentication**, where each user action confirms their right to operate by building upon the result of prior actions.

The core idea behind ULDA is that, at each step, the system must:

- confirm **ownership of the previous state** (`ancestor`),
    
- provide **a commitment to the next state** (`descendant`),  
    thereby forming a continuous chain of authenticated actions.
    

This structure can be visualized as a **directed signature ladder**:

`x₁*₀ → x₂*₁ → x₃*₂ → x₄*₃ → ...`

Where:

- `x₁*₀` — the initial (master) key or signature,
    
- `x₂*₁` — the next step, used to confirm the first one (`x₂*₀`),
    
- and so on.
    

In practice, this means that when a user performs an operation, they must provide:

- **proof of the previous state** (e.g., `x₂*₀` to confirm `x₁`),
    
- **and a preview of the next** (`x₃*₁`), which becomes part of the next authentication step.
    

Thus, unlike traditional tokens or JWTs, ULDA does not require constant synchronization with the server, assumes no stored session state, and operates in a context of **zero trust** between parties.

---

## Inheritance

Inheritance in ULDA means that each subsequent state is logically and cryptographically dependent on the previous one. However, ULDA also offers **flexible verification modes**, allowing the complexity of proof to be adjusted:

### 1. **Basic Linear Mode**

Each action confirms only the immediately preceding one:

- `x₁` → `x₂`, `x₂` → `x₃`, and so on;
    
- Requires one signature and one forward preview;
    
- The minimal requirement: `x_{n+1}*n` confirms `x_n`.
### 2. **Extended Mode with Multiple Signatures**

To prevent signature reuse or forgery, it is possible to define that:

- Verification requires `n` signatures instead of just one.
    
- Example: updating a file may require the client to present `5` valid signatures from the previous chain (or branch).
    
- This allows for flexible control over the level of trust and resistance to attacks.
    

### 3. **Skip-proof Mode**

Occasionally, it may be necessary to skip a step (e.g., due to data loss or chain recovery):

- ULDA allows **skipping one or more steps**, as long as enhanced proof is provided.
    
- For example, `x_{n+3}*n` may be used, in which case the system demands additional signatures or a special range signature (akin to a linear ZKP).
    
- This mode is particularly valuable in unstable network environments or during partial client failure.
    

### 4. **Asynchronous Branching**

The system can support parallel flows (`xₙₐ`, `xₙᵦ`) when it is necessary to maintain multiple “chains” under one user or one file.  
This enables:

- working with multiple records concurrently,
    
- synchronizing changes without conflict,
    
- managing concurrent updates through controlled signature verification.
    

---

**Inheritance makes ULDA not only an authentication mechanism, but also a form of cryptographic version control**, in which each modification requires proof of the previous step and preparation of the next. This is the fundamental logic that underlies the entire ULDA architecture and its practical implementation in 0am.

## Variants of Solutions

### 1. Basic Linear Mode

The simplest form of ULDA is built on a sequential step-by-step structure:

$x_{n} = \mathrm{Sign}\bigl(\;H(\,x_{n-1}\;\| \;x_{n+1}^{\,\mathsf{preview}}\,)\bigr)$

Where:

- $x_{n}​$​ — the current signature at step nnn;
    
- $x_{n-1}$​​ — the previous step being verified;
    
- $x_{n+1}^{\,\mathsf{preview}}$​ — a public preview of the next signature, attached to the current step’s message;
    
- $H(\cdot)$ — a cryptographic hash function;
    
- $Sign(⋅)$ — a symmetric or asymmetric signature function.
    

In practice, the client sends the pair  
$\bigl\langle x_{n-1},\,x_{n+1}^{\,\mathsf{preview}}\bigr\rangle$   
the server checks that $x_{n-1}$​ matches the previously published $x_{n-1}^{\,\mathsf{preview}}$​, and stores $x_{n+1}^{\,\mathsf{preview}}$​​ as the expected proof for the next step.

---

### 2. Extended Mode with Multiple Signatures

To increase resistance to forgery and replay attacks, ULDA allows for requiring multiple "layers" of signatures. Conceptually, this is implemented as a **pyramidal ladder**, where each node at step iii contains a cascade of hashes of its ancestors.

In 0am, this is built using the `generateLinkedHashes` function (simplified, without CRC details):

$Let  ​Li(0)​=seedi​,Li(k)​=H(Li(k−1)​),$

$k=1…di​,xi​=Li(di​)​,$

$di​=i−start.$

- The algorithm iterates through all indices i from `start` to `end`;
    
- For each step, it builds a **chain of length $d_i$​​**, where each layer is a hash of the previous one;
    
- The final signature xix_ixi​ is equal to the last layer $L_i^{(d_i)}$​;
    
- The further a step is from the start, the greater its depth $d_i$​, making forgery significantly harder without full chain knowledge.
    

**Comment:** This process forms a triangular dependency matrix:  
the top row contains single-layer signatures, while the bottom holds the deepest cascades.

---

### 3. Skip-proof (Proof of Skipping)

ULDA allows “jumps” over one or more steps if the client provides enhanced proof. Let’s consider two representative cases:

| Scenario                          | Step Formula $n{+}\Delta$              | Is Skipping Allowed? | Reason                                                                                                                                                                                                                                   |
| --------------------------------- | ------------------------------------------ | -------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Pure Vertical Dependency**      | $\boxed{x_{44}=H\bigl(x_{43}\bigr)}$​      | **Yes**              | To verify $x_{44}$​, it is sufficient to present $x_{43}$​. If a signature was lost, the client can send $x_{44}​ + x_{43}$​ and skip over the missing $x_{42}$.                                                                         |
| **Mixed (Horizontal + Vertical)** | $\boxed{x_{44}=H\bigl(x_{43}x_{33}\bigr)}$ | **No**               | Verification requires **two** components — $x_{43}$​ _and_ $x_{33}$​. Skipping steps would require computing $H^{-1}$, which is cryptographically infeasible. As a result, skip-proof is disabled, but overall security is strengthened. |

**Formal Rule:**  
Let $x_{n+\Delta}=H\bigl(\mathcal{C}\bigr)$, where $\mathcal{C}\subseteq\{x_{k}\}_{k\le n+\Delta-1}$.

- If $\mathcal{C}|=1$ and $\mathcal{C}=\{x_{n+\Delta-1}\}$ (pure vertical) ⇒ skipping is allowed by presenting $\mathcal{C}$;
    
- If $\mathcal{C}|>1$ or $\mathcal{C}$ includes non-adjacent indices ⇒ skipping would require computing $H^{-1}$ on at least one argument ⇒ practically infeasible ⇒ skip-proof is disabled.
    

This gives developers control over the trade-off between:

- **Recovery flexibility** — achievable with vertical forms like $x_{k+1}=H(x_k)$;
    
- **Maximum robustness** — ensured by including additional elements (e.g., $x_{k-1}, x_{k-11}$, etc.) into the hash argument.
    

---

### 4. Asynchronous Branching

In theory, ULDA supports multiple parallel “branches” $x_{n,a}, x_{n,b}$, similar to forks in version control systems, but with cryptographic merge verification.

- The model requires a consistent branch ID dictionary and well-defined merge-proof rules;
    
- In 0am, this feature is marked as **experimental**: implementation is planned after formalizing the conflict resolution protocol.
    

---

### 5. Backup Signatures for Skip-proof

To ensure recovery after a long skip $(\Delta n \ge n_{\text{threshold}})$, ULDA uses **backup signatures**:

- For each step, a pair is generated:  
    $\langle x_{n},\,b_{n}\rangle$ , 
    where $b_{n}$​ is stored offline or on an independent node;
    
- If the chain is lost, the client can present $b_{k}$​, proving ownership of block `k` and initiating a new ladder from that point;
    
- This mechanism is already implemented in 0am and fully integrated with skip-proof logic.
## Signature Packaging (0am)

In 0am, each ULDA "ladder" is serialized into a compact binary container, transferred between client and server as a unified **Signature Package (SP)**. Below is the current field specification and its purpose:

|Order|Field (Nickname)|Size|Purpose|
|---|---|---|---|
|1|**Height** (`HideHt`)|1 B|Line length `N` (0–255). Recommended value is 5: allows skipping up to four steps without backup.|
|2|**Age** (`Age`)|4 B|Sequence number of the **first** signature in the line. Helps the server determine where to start verifying.|
|3|**Line** (`Line`)|24 B × `N`|The ULDA signature line itself. Each signature takes 24 bytes (truncated hash + CRC layer).|
|4|**SolidBack**|16 B|Reference backup signature. If the server detects a mismatch → triggers recovery procedure.|
|5|**AddedBack**|32 B|Line used to verify `SolidBack` and allow reconstructing the chain.|
|6|**BackupAge**|1 B (default)|Inverted value of the gap (`Δn`). Lower value means the backup is closer to current state; max = 254.|

**Total container size:**

${Size} = \begin{cases} 21 + 24N, & \text{without backup block} \\ 54 + 24N, & \text{with backup block} \end{cases}$,​without backup blockwith backup block​

_Typical values:_  
`N = 5` → 141 B (basic) or 174 B (with backup)

---

### Validation Rules and "Quick Jumps"

- **Quick Jump** is allowed if the actual gap $Δn≤Height−1$.  
    _Example:_ with `N = 5`, a client may skip up to 4 steps while still confirming a line in a single packet.
    
- **Backup mode** is triggered when $\Delta n > 5$.  
    The server requests both `SolidBack` and `AddedBack`, and verifies consistency via `BackupAge`. The maximum number of attempts is governed by `BackupAge`; after the limit is exceeded, the client should create a new Master File.
    
- **Failure** — if backups are depleted and the `gap` continues to grow, the connection is deemed untrusted. The client is advised to export the data and regenerate the chain.
    

---

### Summary

- **Compactness.** In the worst case: 174 bytes per line — ideal for low-bandwidth networks.
    
- **Self-sufficiency.** One packet contains everything needed to verify past actions **and** announce the next.
    
- **Recovery flexibility.** `SolidBack` + `AddedBack` allow repairing the chain without data loss, if the skip stays within bounds.
    

---

# Use Cases

## Zero Trust

Traditional authentication requires the server to store a secret — a password hash, private key, token salt, etc.  
**ULDA reverses this model**: the server only needs to store an **autonomous evolving signature chain**, which is exponentially harder to spoof than a basic hash — and the **password never leaves the client device in any form**.

| How the "no secrets" problem is solved                                                  | Clarification                                                                               |
| --------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------- |
| 1. Client ▸ sends a valid segment of the chain $⟨x_n , x_{n+1}^{preview}⟩$.             | $x_n$​ proves ownership of the previous state; $x_{n+1}^{preview}$ announces the next step. |
| 2. Server ▸ verifies that xnx_nxn​ is exactly what was expected after the last request. | Only the _expected_ signature is stored in server memory — never a secret.                  |
| 3. Server ▸ saves the new “expected” value ($x_{n+1}^{preview}$) — and nothing else.    | Even full server compromise **won’t expose** any password or future signature.              |

Thus, the user proves: **“I am the same entity that made the previous step.”**  
This historical proof of identity effectively replaces passwords / cookies / JWTs in systems where storing secrets is prohibited.

---

#### Resilience to MITM Attacks

_A MITM intermediary may eavesdrop or even try to place itself "in the middle."_

1. **Interception scenario.** An attacker captures $⟨x_n , x_{n+1}^{preview}⟩$.
    
2. **Only viable move** — instantly replay that packet to the server, hijacking a single request.
    
3. **Outcome:**
    
    - Immediately afterward, the server expects a **new** $x_{n+1}$​ from the legitimate client;
        
    - The original client detects a mismatch and re-issues the chain (via backup/skip-proof);
        
    - The session either breaks here, or is restored after one step — and the stolen action becomes worthless as the chain moves forward.
        

If the payload is additionally **encrypted**, the attacker learns nothing of the contents, and the hijacked step provides no path to computing the next — since the attacker lacks both the password and full history.

**Conclusion:** The **ULDA “prove-you-were-here-before” model** replaces stored secrets and makes MITM attacks self-limiting: the channel either breaks immediately or repairs itself after a single unauthorized request.

---

## 0am as a Use Case

**0am** is a SaaS platform built around the idea of **universal client-side encryption**, where ULDA serves as the **core authentication and access control mechanism**.  
Instead of relying on stored tokens, passwords, or private keys, the system uses **inheritable signatures**, implemented via ULDA.

In practice, this means:

- The user receives a **JavaScript library** that runs in the browser or client-side app;
    
- All data is encrypted **on the client**, before ever reaching the server;
    
- Each operation (create, update, delete) requires proof of the previous signature and generates the next;
    
- Authentication is embedded in the structure of the data itself — no need for login or sessions.
    

Thanks to this approach, **0am makes client-side encryption simple, scalable, and truly accessible** — both technically and economically.

---

#### Highlights of the 0am Architecture:

- **Cryptography without server load** — all access control is managed client-side; the server is a passive storage backend;
    
- **Plug-and-play integration** — just import the library and send data in the expected format;
    
- **Built-in recovery mechanisms (backup, skip-proof)** — even with data loss, the client can recover authority over their content;
    
- **Extensible signature model** — from simple linear steps to deep hierarchical signature ladders for critical operations.
    

In short, 0am is not just an implementation of ULDA — it is a **practical demonstration of how cryptography can be transparent and cost-effective**, without compromising security.

---

# Limitations

Despite ULDA’s strengths — especially its freedom from stored secrets and resilience to attacks — this approach has important limitations, particularly when evaluated as a universal solution.

## Key Dependency and Chain Management

When ULDA is used in its "pure" form — without enhancements like the master password and chain encryption seen in 0am — the client must **independently store and manage their signature chain**:

- preserve the current state (last signature and expected next one);
    
- ensure integrity and availability of this data across sessions;
    
- protect the chain from compromise.
    

In web or mobile applications with limited storage and cross-device synchronization, this can become a **non-trivial challenge**, especially outside a controlled application environment.

---

## Integration Complexity

ULDA is a **non-standard and radical model** that differs significantly from traditional approaches (login, token, cookie).  
Integrating ULDA often requires:

- rethinking client-server architecture;
    
- abandoning conventional session-based authorization;
    
- rewriting existing APIs around the "signature + prediction" model.
    

As a result, ULDA is often best adopted **in new applications** designed specifically for it.  
Retrofitting ULDA into existing systems may demand **substantial changes and investment**, making it difficult to adopt at scale.

---

## Niche Use Case

The problem that ULDA solves — **authentication without the ability to store secrets on the server** — is inherently **a niche scenario**.

Most modern web applications or services:

- have access to secure storage (e.g., HSM, TPM, enclaves);
    
- use trusted server infrastructure;
    
- or simply don’t require such deep decentralization of authentication.
    

Therefore, while promising, ULDA remains best suited for **specific contexts**, such as:

- public databases with zero-trust guarantees;
    
- cryptographic peer-to-peer protocols;
    
- apps aiming for full client independence from server infrastructure.


