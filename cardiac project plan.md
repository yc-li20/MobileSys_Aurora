
# Multimodal Papers: Contributions and Research Directions

---

## 1. What These Multimodal Papers Actually Contributed

### MISA: Separating Shared and Modality-Private Information

The main contribution of MISA is not simply another fusion block. It projects each modality into two subspaces:

$$
h_m = h_m^{\text{shared}} + h_m^{\text{private}}
$$

The shared space captures information common across modalities, while the private space preserves modality-specific information that may still be useful for the task. MISA uses:

- A **similarity loss** to bring shared representations closer
- An **orthogonality loss** to reduce redundancy between shared and private representations
- A **reconstruction loss** to prevent the private representation from becoming degenerate

#### Relevance to CardioState-JEPA

CardioState-JEPA tries to place ECG, PPG, and PCG into a shared cardiac space. But "shared" does not mean that all useful information should be shared.

ECG conduction abnormalities, PCG murmur morphology, and PPG vascular-waveform features may not be recoverable from the other modalities. MISA's experiments also suggest that using only invariant representations is too restrictive; combining shared and modality-specific representations works better.

#### Research Direction: Shared–Private Cardiac Representation

Instead of learning only one cardiac code, learn:

$$
z_m^{\text{shared}}, \qquad z_m^{\text{private}}
$$

where:

- $z_m^{\text{shared}}$ captures cross-modal cardiac-cycle structure and physiological state
- $z_m^{\text{private}}$ preserves information observable mainly by ECG, PPG, or PCG
- Optionally, $z_m^{\text{task}}$ captures information relevant to a specific clinical task

**The central question becomes:**
> Which cardiac features are genuinely shared across modalities, and which pathological signals must remain modality-specific?

A useful experiment would compare:

| Input Representation | ECG Tasks | PPG Tasks | PCG Tasks |
|---|---|---|---|
| Shared code only | Measure | Measure | Measure |
| Private code only | Measure | Measure | Measure |
| Shared + private | Measure | Measure | Measure |
| Original CardioState-JEPA code | Baseline | Baseline | Baseline |

The most valuable outcome would be findings such as:

- The shared code is useful for **rhythm, heart rate, and cycle structure**
- ECG-private information is important for **conduction and arrhythmia tasks**
- PCG-private information is important for **murmur detection**
- PPG-private information is important for **vascular and blood-pressure tasks**
- Overly strong cross-modal alignment can **damage modality-specific diagnostic information**

This would produce a clear scientific claim:

> **Cross-modal consistency and clinical discriminability are not always aligned.**

---

## 2. MTAG's Lesson: Model Event Relationships, Not Only Global Alignment

MTAG does not simply concatenate modalities. It treats temporal segments from each modality as graph nodes and represents modality relationships and temporal relationships as typed edges. The model learns which cross-modal edges are useful and which should be removed.

**The key lesson is:**
> For asynchronous physiological signals, you may not want to first estimate a single $\tau$ and align everything according to that delay. You could instead model which events are related to one another.

CardioState-JEPA currently uses a continuous delay:

$$
t_{\text{target}} = t_{\text{source}} + \tau
$$

However, one ECG R-peak may correspond to:

- PCG's S1
- The PPG upstroke
- The PPG systolic peak at a *different* delay
- Multiple events within one cardiac cycle
- **Different event relationships under pathological conditions**

#### Cardiac Event Relation Graph

**Nodes** could include:

| Modality | Events |
|---|---|
| ECG | P, QRS, T events |
| PCG | S1, S2, murmur bursts |
| PPG | Upstroke, systolic peak, dicrotic notch |
| Beat-level | Latent cardiac states |

**Edges** could represent:

- $\text{ECG} \rightarrow \text{PCG}$
- $\text{ECG} \rightarrow \text{PPG}$
- Within-beat relationships
- Previous-beat and next-beat relationships
- Electrical-to-mechanical coupling
- Electrical-to-hemodynamic coupling

The model would then dynamically select important edges. In MTAG, adding modality and temporal edge types improved performance, and attention-based edge pruning outperformed random pruning, suggesting that spurious cross-modal relations can distract the model.

**This leads to a stronger research question:**
> Should relationships among cardiac signals be modeled as event-level, directional, and state-dependent relations rather than as one fixed continuous delay?

The novelty would be:

- Moving from **waveform alignment** to **physiological event relations**
- Replacing **one delay** with **multiple event-level relationships**
- Learning **pathological-state-dependent** relations
- Replacing opaque latent similarity with an **interpretable cardiac event graph**

---

## 3. Incongruity-Aware Fusion: Cross-Modal Information Can Be Harmful

*Cross-Attention is Not Enough* is particularly relevant. It argues that cross-attention can help one modality enhance another, but modalities may also contain inconsistent or even opposing signals. Unconditionally fusing them can introduce redundancy and misleading information.

In cardiac sensing, incongruity could mean:

- ECG appears regular while PCG contains a murmur
- PPG is corrupted by motion while ECG remains clean
- The ECG–PPG timing relationship changes
- PCG contains environmental noise
- One modality reflects pathology while another modality has simply failed

#### Physiological Incongruity-Aware Fusion

The model should distinguish among:

1. Normal cross-modal differences
2. Sensor noise
3. Pathological physiological inconsistency
4. Modality differences that are useful for the current task

Define an event-level incongruity score:

$$
r_{mn}(t) = d!\left( z_m(t),\ \mathrm{Align}(z_n, t) \right)
$$

However, the goal should **not** simply be to minimize $r_{mn}(t)$. Instead, the model could learn task-dependent modality weights:

$$
\alpha_m(t) = f\bigl( q_m(t),\ r_{mn}(t),\ y_{\text{task}} \bigr)
$$

where:

- $q_m(t)$ is the signal quality
- $r_{mn}(t)$ is the cross-modal inconsistency
- $\alpha_m(t)$ is the modality weight for the current task

The model could then learn that:

- **ECG–PCG disagreement** may be useful for murmur detection
- When **PPG is corrupted by motion**, ECG should receive more weight for rhythm prediction
- For **pulse-transit-time estimation**, ECG–PPG delay changes may be the target rather than noise

**The important conceptual point is:**
> Modality invariance should not always be the ultimate objective. Some cross-modal differences may themselves be diagnostic signals.

CardioState-JEPA reports that the modality silhouette decreases from $0.121$ after Stage I to $-0.006$ after Stage II, indicating that the representation becomes substantially more modality-invariant.

This demonstrates successful modality alignment, but it does not answer:

> After modality-specific separation is removed, has the model also removed diagnostically useful differences?

That is a strong opening for your work.

---

## 4. FMT's Lesson: Do Not Mix All Interactions into One Attention Mechanism

Factorized Multimodal Transformer explicitly separates:

- **Unimodal interactions**
- **Bimodal interactions**
- **Trimodal interactions**

Rather than asking one generic self-attention mechanism to discover every relationship, it assigns different attention factors to different modality combinations. The paper argues that this factorization can model both intramodal and intermodal dynamics while handling long-range asynchronous relationships.

#### Physiological Interpretation

For ECG, PPG, and PCG, these interaction levels have distinct physiological meanings:

| Level | Meaning |
|---|---|
| **Unimodal** | P-QRS-T in ECG; upstroke-to-systolic-peak in PPG; S1/S2 in PCG |
| **Bimodal** | ECG–PPG pulse arrival time; ECG–PCG electromechanical timing |
| **Trimodal** | Mutual consistency of electrical, mechanical, and hemodynamic signals within a cycle |

You could therefore define:

$$
z_m^{\text{out}} = F_m^{\text{intra}} + F_{m,n}^{\text{pair}} + F_{m,n,k}^{\text{tri}}
$$

**The key question would be:**
> Which tasks require unimodal, pairwise, or trimodal interactions?

This is more informative than simply asking whether three modalities outperform one modality.

CardioState-JEPA's ablation results already suggest that the third modality does not help all tasks equally: the trimodal model is especially beneficial for **PPG regression** and **PCG**, while ECG performance is close to the best bimodal variant.

---

## 5. Two Concrete Research Directions

### Direction A: Shared–Private–Relational Cardiac Representation

This is probably the most natural extension of CardioState-JEPA. The model would learn three types of information:

$$
z_m = \underbrace{z^{\text{shared}}_m}_{\text{cross-modal common state}} + \underbrace{z^{\text{private}}_m}_{\text{modality-specific information}} + \underbrace{z^{\text{rel}}_{m,n}}_{\text{cross-modal relations}}
$$

**Possible training objectives:**

- Cross-modal latent prediction for the **shared code**
- Modality-specific masked prediction for the **private code**
- Event-level timing and relation objectives for the **relational code**
- Task-adaptive gating over the three types of representation

**The main experiment** should be an information decomposition study:

| Condition | Components Used |
|---|---|
| 1 | Shared only |
| 2 | Private only |
| 3 | Relational only |
| 4 | Shared + private |
| 5 | Shared + relational |
| 6 | All components |

If different clinical tasks depend on different components, your paper would answer a deeper question:

> **How is information distributed across multimodal cardiac signals?**

---

### Direction B: Task-Adaptive Physiological Incongruity Modeling

This direction is more focused on **clinical reliability and dynamic fusion**. The model would estimate:

- Signal quality for each modality
- Cross-modal alignment uncertainty
- Physiological incongruity
- Task-specific modality utility

The output would include both a prediction and uncertainty:

$$
p(y \mid x_{\text{ECG}},\ x_{\text{PPG}},\ x_{\text{PCG}}) \quad\text{and}\quad u_{\text{modal}}
$$

You could evaluate four conditions:

| Condition | Description |
|---|---|
| 1 | Normal synchronization |
| 2 | Artificial temporal offsets |
| 3 | Sensor corruption in one modality |
| 4 | Genuine pathological timing abnormalities |

**The ideal behavior would be:**

- **Downweight** a modality under sensor corruption
- **Preserve or increase** its importance when the inconsistency is pathological
- Treat **abnormal ECG–PPG timing** as a meaningful signal rather than merely an alignment error

This turns cross-modal consistency from a simple training constraint into **a variable that must be interpreted carefully**.

---

## 6. From Research Directions to a Foundation Model: Any-Modality Cardiac Foundation Model

### 6.1 Why the Two Directions Above Are Not Yet a Foundation Model

Direction A and Direction B described above are meaningful research contributions, but they do not independently constitute a foundation model. A foundation model requires three properties:

- **Scale**: pretrained on large amounts of unlabeled data
- **Generality**: one model serves multiple downstream tasks without architectural changes
- **Emergence**: downstream tasks only require fine-tuning or probing, not redesign

Direction A (shared–private decomposition) addresses representation structure. Direction B (incongruity-aware fusion) addresses inference-time reliability. Neither addresses the fundamental question of **what happens when a modality is missing at inference time** — which is the most common real-world scenario.

This is the gap that motivates the Any-Modality Cardiac Foundation Model.

---

### 6.2 CardioState-JEPA's Actual Training Procedure

Before identifying what JEPA left open, it is important to understand what it actually does — because the two-stage training design is the source of both its strengths and its gaps.

#### Stage I: Within-Modality Pretraining (large unimodal datasets)

Stage I trains entirely on **single-modality data**. Each modality is fed independently into the shared Transformer encoder, which learns to predict **masked latent cardiac states** within that modality:

```
Stage I (unimodal, large datasets):
  ECG alone  → predict masked ECG latent states
  PPG alone  → predict masked PPG latent states
  PCG alone  → predict masked PCG latent states
```

The key motivation is data availability: synchronized multi-sensor recordings are scarce, but single-modality cardiac data is abundant. Stage I exploits this abundance to learn within-modality structure before any cross-modal signal is introduced.

This means Stage I already supports single-modality inference — but only because each modality is trained independently, not because the model is explicitly designed to handle missing modalities.

#### Stage II: Cross-Modal Alignment (scarce paired data)

Stage II uses **synchronized multi-modality recordings** to align the three modalities in latent cardiac time. Because the three signals observe the same beat at different physiological delays (PCG follows electromechanical coupling; PPG arrives later via pulse transit time), a **learned delay aligner** is used to match signals at the corresponding cardiac time before cross-modal latent prediction:

```
Stage II (multimodal, small paired datasets):
  ECG + PPG + PCG (synchronized) →
    delay aligner matches signals at corresponding cardiac time →
    cross-modal masked latent prediction
```

The pretraining target is placed on **shared physiology** rather than sensor-specific waveform appearance, which is the core JEPA idea applied to the cross-modal setting.

#### What This Two-Stage Design Implies

- Single-modality inference at test time is **inherited from Stage I**, not explicitly designed
- Stage II requires all three modalities to be simultaneously available in the training data
- There is no training signal for the case where only a subset of modalities is available at **Stage II time** — the model never sees "ECG + PPG but no PCG" during cross-modal training
- The delay aligner assumes a fixed temporal offset structure; it does not model state-dependent or pathology-dependent changes in cross-modal timing

---

### 6.3 What JEPA Left Open

| Property | CardioState-JEPA | Any-Modality Foundation Model |
|---|---|---|
| Stage I objective | Within-modality masked latent prediction (single modality input) | Within-modality self-reconstruction; explicitly learns $z^{\text{private}}_m$ |
| Stage II objective | Cross-modal alignment on synchronized three-modality data | Bidirectional cross-modal reconstruction on **any modality subset**, including pairs |
| Stage II data requirement | Requires all three modalities synchronized | Can exploit two-modality paired data (ECG+PPG, ECG+PCG, etc.) |
| Representation structure | Single shared cardiac code | Shared + private decomposition |
| Missing modality handling | Stage I ability only — no cross-modal training for partial subsets | Explicit random modality masking during Stage II; $z^{\text{shared}}$ estimated from any subset |
| Missing modality compensation | None — falls back to unimodal Stage I representation | $z^{\text{shared}}$ reconstructed from remaining modalities; $z^{\text{private}}_m$ explicitly marked absent |
| Cross-modal information flow | Alignment only (pull representations closer in latent space) | Bidirectional reconstruction (information explicitly flows across modalities) |
| Cross-modal timing model | Fixed learned delay $\tau$ per modality pair | Event-level relations; timing changes under pathology are preserved, not corrected |
| Modality incongruity | Not handled — cross-modal differences are treated as alignment error | $\alpha_m(t)$ distinguishes sensor noise, pathological incongruity, and modality absence |

The core argument is:

> JEPA's Stage I gives it single-modality capability as a byproduct of data efficiency, not as a design goal. Stage II only trains on complete three-modality synchronized data, so the model never learns during pretraining how to operate when one or two modalities are absent. This framework makes missing-modality robustness a **first-class pretraining objective**, and can additionally exploit the much larger pool of two-modality paired data that JEPA's Stage II cannot use.

---

### 6.4 How Each Prior Work Connects to This Framework

#### MISA → Core Representation (Direct, Essential)

MISA's shared–private decomposition is the **structural foundation** of the any-modality framework. When modality $m$ is missing, the framework needs to know:

- What can be estimated from other modalities? → $z^{\text{shared}}$
- What is permanently unavailable? → $z^{\text{private}}_m$

Without this decomposition, missing-modality inference has no principled basis. The shared space can be estimated from any available subset; the private space is explicitly marked as absent.

$$z_m = z^{\text{shared}}_m + z^{\text{private}}_m$$

In the any-modality setting, this becomes:

$$z^{\text{shared}} = \text{Aggregate}\!\left(\{z^{\text{shared}}_m\}_{m \in \mathcal{M}}\right), \qquad z^{\text{private}}_m = \begin{cases} \text{encoded} & m \in \mathcal{M} \\ \mathbf{0} \text{ or } \texttt{[MISSING]} & m \notin \mathcal{M} \end{cases}$$

#### Incongruity-Aware Fusion → Inference Mechanism (Direct, Unified)

In the original Direction B, incongruity modeling handled three cases: normal signal, sensor corruption, and pathological inconsistency. In the any-modality framework, a **fourth case** is added naturally:

$$\alpha_m(t) = \begin{cases} f\!\left(q_m(t),\ r_{mn}(t)\right) & \text{modality present, normal quality} \\ f\!\left(q_m(t),\ r_{mn}(t)\right) \to 0 & \text{modality present, corrupted} \\ \text{(diagnostic signal)} & \text{modality present, pathological incongruity} \\ 0 & \text{modality absent} \end{cases}$$

Modality absence becomes a **special case of zero weight**, not a separate code path. This unification means the model handles missing modalities and corrupted modalities through the same mechanism, which is both architecturally cleaner and clinically more realistic.

#### FMT → Architecture Backbone (Indirect, Engineering)

FMT's factorized attention provides a natural architecture for the any-modality setting because the interaction levels degrade gracefully when modalities are absent:

$$z_m^{\text{out}} = F_m^{\text{intra}} + \sum_{n \neq m,\ n \in \mathcal{M}} F_{m,n}^{\text{pair}} + F_{m,n,k}^{\text{tri}} \cdot \mathbf{1}[\,|\mathcal{M}| = 3\,]$$

When only one modality is present, only $F_m^{\text{intra}}$ is active. When two are present, pairwise interactions are added. When all three are present, trimodal interactions engage. This is not a novel contribution on its own, but it is the right architectural choice for this framework.

#### MTAG → Interpretability Analysis (Indirect, Supporting)

MTAG's graph structure is **not recommended as a primary module** in this framework, because the graph topology changes as modalities are added or removed, which adds complexity without clear benefit to the core any-modality objective.

However, MTAG's insight about typed, directional edges is valuable for **post-hoc interpretability**: once the model is trained, a cardiac event graph can be used to visualize which cross-modal relationships the model relies on under different modality subsets. This is useful for clinical trust and for understanding what the model has learned, but it is analysis rather than architecture.

---

### 6.5 The Pretraining Objective: Bidirectional Reconstruction + Consistency

The pretraining objective combines two goals acting on different parts of the representation:

$$\mathcal{L} = \underbrace{\sum_{m,n \in \mathcal{M},\ m \neq n} \mathcal{L}_{\text{recon}}(m \to n)}_{\text{bidirectional cross-modal reconstruction}} + \underbrace{\lambda_1 \sum_{m,n \in \mathcal{M}} \mathcal{L}_{\text{align}}\!\left(z^{\text{shared}}_m,\ z^{\text{shared}}_n\right)}_{\text{shared space consistency}} + \underbrace{\lambda_2 \sum_{m \in \mathcal{M}} \mathcal{L}_{\text{ortho}}\!\left(z^{\text{shared}}_m,\ z^{\text{private}}_m\right)}_{\text{prevent information leakage}}$$

The three losses are designed to not conflict with each other:

| Loss | Acts On | Prevents |
|---|---|---|
| $\mathcal{L}_{\text{recon}}$ | shared + private → target modality latent | $z^{\text{private}}_m$ collapsing to zero |
| $\mathcal{L}_{\text{align}}$ | shared space only | shared representations diverging across modalities |
| $\mathcal{L}_{\text{ortho}}$ | shared ↔ private | private information leaking into shared space |

#### Why Bidirectional and Not Unidirectional?

Unidirectional reconstruction (mask one modality, reconstruct from others) is simpler but asymmetric. It does not force the model to learn that ECG and PPG carry complementary information about the same cardiac event — only that one can be predicted from the other. Bidirectional reconstruction forces the model to simultaneously maintain both directions, which produces a more symmetric and complete shared space.

#### Two Pitfalls of Bidirectional Reconstruction and Their Solutions

**Pitfall 1 — Information shortcut**: the model may route all information through $z^{\text{shared}}$ to minimize reconstruction loss, making $z^{\text{private}}_m$ redundant. This is prevented by $\mathcal{L}_{\text{ortho}}$, which penalizes overlap between the two subspaces.

**Pitfall 2 — Symmetry collapse**: if both directions use the same decoder, the model may map all modalities to the same point. This is prevented by using **modality-specific decoders** and **stop-gradient** on the target:

$$\mathcal{L}_{\text{recon}}(m \to n) = \mathcal{L}\!\left(f_n^{\text{dec}}\!\left(z^{\text{shared}}_m,\ z^{\text{private}}_m\right),\ \text{sg}(z_n)\right)$$

where $\text{sg}(\cdot)$ denotes stop-gradient, borrowed from BYOL and JEPA to prevent collapse without requiring negative samples.

#### Single-Modality Case: Self-Reconstruction

When $|\mathcal{M}| = 1$, the cross-modal terms in $\mathcal{L}_{\text{recon}}$ and $\mathcal{L}_{\text{align}}$ are empty. A self-reconstruction objective ensures that $z^{\text{private}}_m$ still carries meaningful content:

$$\mathcal{L}_{\text{self}} = \mathcal{L}\!\left(g_m^{\text{dec}}\!\left(z^{\text{shared}}_m,\ z^{\text{private}}_m\right),\ \text{sg}(z_m)\right)$$

This creates a natural curriculum across modality counts:

$$\underbrace{\text{single-modality self-reconstruction}}_{\text{learn private space}} \;\to\; \underbrace{\text{bimodal cross-reconstruction}}_{\text{learn cross-modal relations}} \;\to\; \underbrace{\text{trimodal full objective}}_{\text{learn global consistency}}$$

---

### 6.6 Training: Modality Sampling Strategy

A critical implementation detail is that **the model must not always train on all three modalities**. If trimodal training dominates, the model learns to rely on full modality availability and degrades under missing-modality inference.

Recommended sampling distribution per batch:

$$p(\mathcal{M}) \propto \begin{cases} 2 & |\mathcal{M}| = 1 \\ 2 & |\mathcal{M}| = 2 \\ 1 & |\mathcal{M}| = 3 \end{cases}$$

Oversampling low-modality cases forces the any-modality capability to genuinely emerge rather than being a post-hoc adaptation.

---

### 6.7 Inference: Modality-Adaptive Aggregation

At inference time, the shared representation is estimated by aggregating across all available modalities:

$$z^{\text{shared}} = \text{Aggregate}\!\left(\{z^{\text{shared}}_m\}_{m \in \mathcal{M}}\right)$$

The aggregation weights are determined by the incongruity-aware modality weights $\alpha_m(t)$ from Section 3, which naturally handle the full spectrum from clean signals to corrupted inputs to absent modalities.

The final representation passed to any downstream task is:

$$z^{\text{final}} = \left[z^{\text{shared}},\ \{z^{\text{private}}_m\}_{m \in \mathcal{M}},\ \{\texttt{[MISSING]}\}_{m \notin \mathcal{M}}\right]$$

A downstream task head learns which components are relevant for its specific clinical objective, rather than receiving a single fixed cardiac code.

---

### 6.8 Complete Framework Overview

```
Pretraining
│
├── Input: randomly sampled modality subset M
│   └── sampling bias: p(|M|=1) : p(|M|=2) : p(|M|=3) = 2 : 2 : 1
│
├── Encoder (per modality, independent)
│   └── x_m → Encoder_m → [z^shared_m | z^private_m]
│
├── Training Objectives
│   ├── Self-reconstruction (single-modality case)
│   │     z^shared_m + z^private_m → sg(z_m)
│   ├── Bidirectional cross-modal reconstruction
│   │     z^shared_m + z^private_m → sg(z_n),  for all m ≠ n in M
│   ├── Shared space consistency
│   │     z^shared_m ≈ z^shared_n,  for all m, n in M
│   └── Orthogonality constraint
│         z^shared_m ⊥ z^private_m
│
Inference
│
├── Available modalities: any subset M' ⊆ {ECG, PPG, PCG}
├── Shared space: aggregated from {z^shared_m}_{m in M'}
├── Private space: present for m in M', [MISSING] for m not in M'
├── Modality weights: α_m(t) from incongruity-aware fusion
└── Downstream head: learns task-specific combination of shared + private
```

---

### 6.9 The Central Scientific Claim

This framework produces a claim that is both broader and more specific than JEPA's:

> **A cardiac foundation model should not require a fixed modality set. Cross-modal consistency and modality-specific discriminability serve different clinical purposes and must be explicitly separated. Modality absence, sensor corruption, and pathological incongruity are points on a single continuum, not separate problems.**

The key empirical question that validates this claim is:

> Does performance degrade gracefully as modalities are removed, and does the model correctly distinguish between missing modalities, corrupted modalities, and pathologically inconsistent modalities?

If yes, the model is genuinely more useful in real clinical deployment than a fixed-modality approach — regardless of whether it achieves higher average AUROC on benchmark datasets.
