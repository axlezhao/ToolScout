# ToolScout: A Small Trained Judge

*Status: plan for an optional research track after v0.1 (M6). Nothing here is a result. Drafted October 25, 2026.*

## 1. Why this exists

The end-to-end evaluation ([EVALUATION.md](EVALUATION.md)) decides whether each agent run succeeded with a large LLM judge (LiveMCPEval). The plan already lists "the judge is wrong" as a risk, and checks it against about 100 hand labels.

This track asks whether a **small judge trained on ToolScout's own agent traces** can do that job as well as the large judge, and whether each training stage helps. The three stages are:

1. **Self-supervised learning (SSL)** on unlabeled traces
2. **Supervised fine-tuning (SFT)** on labeled traces
3. **Reinforcement learning (RL)** with the verdict as a checkable reward

The judge is used only as an **evaluator**: it grades finished runs. It never runs inside the agent loop. A judge that both helps the agent and grades it would bias the measurement, so an in-loop verifier would be a separate study.

## 2. Notation

| Symbol | Meaning |
|---|---|
| $q$ | A task (the user request) |
| $\tau$ | A trace: one recorded agent run for $q$ |
| $a_j$, $o_j$ | The $j$-th tool call in $\tau$ and its result |
| $x = (x_1, \dots, x_T)$ | $\tau$ rendered as text and tokenized |
| $y \in \lbrace \text{success}, \text{fail} \rbrace$ | A verdict |
| $y^{h}$, $y^{s}$ | Gold (human) and silver (large LLM judge) labels |
| $r$ | A short written rationale for a verdict |
| $c(\tau)$ | The condition (A–G) that produced $\tau$ |
| $\pi_\theta$ | The judge model with parameters $\theta$ |
| $\mathcal{D}_U$, $\mathcal{D}_S$, $\mathcal{D}_G$ | Unlabeled, silver-labeled and gold-labeled traces |

**What the judge sees.** The rendered trace contains the task, the calls to real upstream tools, their results and the final answer. It does **not** contain $c(\tau)$, and calls to `search_tools` are removed and `call_tool` calls are unwrapped into the inner tool call. Otherwise the judge could tell ToolScout runs from the others by their shape.

## 3. Hypotheses

Let $\kappa(J)$ be Cohen's kappa between judge $J$ and the gold labels on held-out traces (section 6). The judges compared are:

| Judge | Training |
|---|---|
| $J_{\text{LLM}}$ | The large LLM judge used in M4, unchanged |
| $J_{0}$ | The small base model, prompted, no training |
| $J_{\text{SFT}}$ | Base → SFT |
| $J_{\text{SSL+SFT}}$ | Base → SSL → SFT |
| $J_{\text{SFT+RL}}$ | Base → SFT → RL |
| $J_{\text{full}}$ | Base → SSL → SFT → RL |

**H1. Each stage adds accuracy.**

```math
\kappa(J_{0}) \;<\; \kappa(J_{\text{SFT}}) \;\le\; \kappa(J_{\text{SSL+SFT}}) \;\le\; \kappa(J_{\text{full}})
```

The main claim is that the full pipeline beats SFT alone. It's supported only if the 95% CI of $\kappa(J_{\text{full}}) - \kappa(J_{\text{SFT}})$ lies above 0. The contributions of SSL and RL alone are read from $J_{\text{SSL+SFT}}$ and $J_{\text{SFT+RL}}$.

**H2. A small judge can replace the large one.** For a margin $\delta_2 = 0.05$:

```math
\kappa(J_{\text{full}}) \;\ge\; \kappa(J_{\text{LLM}}) - \delta_2
```

It's supported if the lower bound of the 95% CI of $\kappa(J_{\text{full}}) - \kappa(J_{\text{LLM}})$ is above $-\delta_2$. Cost per verdict is reported next to it.

**H3. The judge doesn't change ToolScout's conclusions.** For two conditions $X$ and $Y$ (the main pair is C vs. D, then each of C and D vs. A), let $\bar s^{J}_{X}$ be the success rate of condition $X$ as graded by judge $J$, and $\bar s^{h}_{X}$ the rate from gold labels. With

```math
\Delta^{J}_{X,Y} = \bar s^{J}_{X} - \bar s^{J}_{Y}, \qquad D_{X,Y} = \Delta^{J}_{X,Y} - \Delta^{h}_{X,Y}
```

H3 holds for $J$ if the 95% CI of $D_{X,Y}$ lies inside $[-\delta_3, \delta_3]$ with $\delta_3 = 0.05$, an equivalence test rather than a failure to find a difference. H3 is checked for both $J_{\text{full}}$ and $J_{\text{LLM}}$.

H3 is the test "through the whole agent": it asks whether ToolScout's end-to-end results would read the same if this judge did the grading.

## 4. The three stages

All stages train LoRA adapters on a small open instruct model (1–3B parameters), and each stage starts from the previous stage's merged weights:

```math
\theta_{0} \xrightarrow{\;\text{SSL}\;} \theta_{\text{SSL}} \xrightarrow{\;\text{SFT}\;} \theta_{\text{SFT}} \xrightarrow{\;\text{RL}\;} \theta_{\text{RL}}
```

A judge that skips a stage starts the next stage from wherever it is (for example, $J_{\text{SFT}}$ runs SFT from $\theta_{0}$).

### 4.1 Stage 1: self-supervised learning on unlabeled traces

Every agent run makes a trace, so unlabeled traces are cheap. Labels are expensive. SSL uses the cheap data to adapt the model to what agent traces look like before it sees any labels. The loss has two parts:

```math
\mathcal{L}_{\text{SSL}}(\theta) =
\underbrace{-\,\mathbb{E}_{\tau \sim \mathcal{D}_U} \sum_{t=1}^{T} \log \pi_\theta(x_t \mid x_{<t})}_{\text{language modeling on trace text}}
\;\underbrace{-\;\lambda\, \mathbb{E}_{\tau \sim \mathcal{D}_U} \sum_{j} \log \pi_\theta\!\left(a_j \mid q,\, a_{<j},\, o_{<j}\right)}_{\text{next tool call prediction}}
```

- The first term adapts the model to the domain: tool names, JSON arguments, error messages.
- The second term predicts each tool call (name and arguments) from everything before it, so the model learns what a sensible next step looks like. A judge that knows that is better placed to spot a run that went off track. Only the tokens of $a_j$ count toward this term.
- $\lambda$ sets the balance between the two terms. It starts at 1 and is chosen on the dev folds (section 5).

Both terms are averaged per token in practice, with the sum normalized by the number of counted tokens.

### 4.2 Stage 2: supervised fine-tuning on labeled traces

Each example is $(q, \tau, y, r)$. The target is the rationale followed by the verdict, so the model writes its reasoning before it decides:

```math
\mathcal{L}_{\text{SFT}}(\theta) = -\,\mathbb{E}_{(q,\tau,y,r) \sim \mathcal{D}_S}\; w_{y} \sum_{k} \log \pi_\theta\!\left(z_k \mid q, \tau, z_{<k}\right), \qquad z = (r, y)
```

- Labels and rationales are silver: they come from the large judge $J_{\text{LLM}}$. Stage 2 is distillation from the large judge into the small one.
- Successful and failed runs won't be equally common, so each class gets a weight:

```math
w_{y} = \frac{|\mathcal{D}_S|}{2\, |\{(q,\tau,y',r) \in \mathcal{D}_S : y' = y\}|}
```

- Only the target tokens $z$ count toward the loss. The prompt (task and trace) doesn't.

### 4.3 Stage 3: reinforcement learning with a checkable reward

SFT can only copy the large judge. Stage 3 moves the small judge toward **human** verdicts, using gold labels as the reward. It uses GRPO (Group Relative Policy Optimization).

For each prompt $p = (q, \tau)$ the current policy samples $G$ outputs $o_1, \dots, o_G$. Each output is scored:

```math
R(o_i) = \mathbb{1}\!\left[\hat y(o_i) = y^{h}\right] \;-\; \mu\, \mathbb{1}\!\left[o_i \text{ has no parsable verdict}\right]
```

where $\hat y(o_i)$ is the verdict parsed from $o_i$ and $\mu = 0.5$ penalizes broken output. Rewards are compared within the group, so no value model is needed:

```math
A_i = \frac{R(o_i) - \operatorname{mean}\{R(o_1),\dots,R(o_G)\}}{\operatorname{std}\{R(o_1),\dots,R(o_G)\} + \varepsilon_{0}}
```

The policy is updated with a clipped objective and a penalty for drifting from the SFT model:

```math
\mathcal{J}_{\text{GRPO}}(\theta) =
\mathbb{E}_{p,\;\{o_i\} \sim \pi_{\theta_{\text{old}}}}
\left[ \frac{1}{G} \sum_{i=1}^{G} \frac{1}{|o_i|} \sum_{k=1}^{|o_i|}
\min\!\Big( \rho_{i,k} A_i,\; \operatorname{clip}(\rho_{i,k},\, 1-\epsilon,\, 1+\epsilon)\, A_i \Big) \right]
\;-\; \beta\, \mathbb{D}_{\text{KL}}\!\left(\pi_\theta \,\|\, \pi_{\theta_{\text{SFT}}}\right)
```

```math
\rho_{i,k} = \frac{\pi_\theta(o_{i,k} \mid p,\, o_{i,<k})}{\pi_{\theta_{\text{old}}}(o_{i,k} \mid p,\, o_{i,<k})}
```

- Starting values: $G = 8$, $\epsilon = 0.2$, $\beta = 0.04$.
- The KL term keeps the judge close to the SFT model, so it can't learn to game the reward with degenerate outputs.
- A group where every output gets the same reward has $A_i = 0$ and teaches nothing. Prompts the SFT model already gets right every time add no signal, which is fine: RL spends its effort on the hard cases.
- Only gold labels are used as rewards. RL on silver labels would only teach the small judge to imitate the large one more closely, so at best it could match $J_{\text{LLM}}$, never move past it toward the human verdicts.

## 5. Data and splits

### 5.1 Pools

| Pool | Source | Planned size | Used by |
|---|---|---|---|
| $\mathcal{D}_U$ | Every M4 trace, plus extra cheap runs (more seeds, other values of $k$) | ~1,000–3,000 traces | Stage 1 |
| $\mathcal{D}_S$ | The same traces with $J_{\text{LLM}}$'s verdict and rationale | Same | Stage 2 |
| $\mathcal{D}_G$ | Hand labels, stratified by condition and by $J_{\text{LLM}}$'s verdict | ~300 traces (up from ~100) | Stage 3 and all testing |

### 5.2 Cross-fitting by task

A judge must never be tested on a task it trained on, even through unlabeled traces. Splitting by trace isn't enough, because runs of the same task look alike.

The task set $\mathcal{T}$ is split into $K = 5$ folds $\mathcal{T}_1, \dots, \mathcal{T}_K$ with a fixed seed. For each fold $k$, a judge $J^{(k)}$ is trained only on traces of the other folds:

```math
J^{(k)} \;\text{is trained on}\; \{\tau : q(\tau) \notin \mathcal{T}_k\}, \qquad
\hat y(\tau) = J^{(k)}(\tau) \;\;\text{for}\;\; q(\tau) \in \mathcal{T}_k
```

Every trace then gets a verdict from a judge that never saw its task, so all ~300 gold labels serve as test data. Hyperparameters ($\lambda$, $\beta$, learning rates) are chosen with an inner split of each fold's training tasks, so the held-out fold is never used for tuning.

Stages are shared between variants (for example, $J_{\text{SSL+SFT}}$ is the starting point of $J_{\text{full}}$), so each fold needs 5 trainings: SSL, SFT from $\theta_0$, SFT from $\theta_{\text{SSL}}$, and RL from each of the two SFT models. Over 5 folds that's 25 trainings, most of them short. That's affordable for a 1–3B model with LoRA.

## 6. Metrics and statistics

**Agreement.** For each judge, on the gold-labeled traces:

```math
\kappa = \frac{p_o - p_e}{1 - p_e}, \qquad
p_o = \frac{1}{n}\sum_{i=1}^{n} \mathbb{1}\!\left[\hat y_i = y^{h}_i\right], \qquad
p_e = \sum_{v \in \{\text{success},\,\text{fail}\}} \hat p(v)\, p^{h}(v)
```

where $\hat p(v)$ and $p^{h}(v)$ are the shares of verdict $v$ from the judge and from the gold labels. Also reported: accuracy, F1 on the `fail` class, and $\kappa$ for each condition separately, to see whether a judge is biased toward one condition.

**Confidence intervals.** Traces of the same task aren't independent, so the bootstrap resamples **tasks**, not traces. For $b = 1, \dots, B$ with $B = 10{,}000$: draw $|\mathcal{T}|$ tasks with replacement, take all their traces, and compute the statistic. For a difference between two judges, both are computed on the same resample:

```math
\widehat{\Delta\kappa}^{(b)} = \kappa^{(b)}(J_{\text{full}}) - \kappa^{(b)}(J_{\text{SFT}}), \qquad
\text{CI}_{95} = \left[\widehat{\Delta\kappa}_{(2.5\%)},\; \widehat{\Delta\kappa}_{(97.5\%)}\right]
```

The same procedure gives the CIs for H2 and for $D_{X,Y}$ in H3.

**Power.** With ~300 gold traces over ~70 tasks, the CI on a paired $\kappa$ difference is expected to be roughly $\pm 0.06$ to $\pm 0.08$. So H1 can detect a stage that adds about 0.08 to $\kappa$, but not a stage that adds 0.02. A smaller gain is reported as "no clear difference", not as a win. The pilot labels (first ~100) are used to check this estimate before labeling the rest.

## 7. Rules for this track

- The judge never sees the condition label, and condition-revealing steps are removed from its input (section 2).
- All judges are frozen before they grade anything that's reported.
- Nobody tunes ToolScout itself using this judge's verdicts. ToolScout's results come from the M4 judge and the gold labels, as in [EVALUATION.md](EVALUATION.md).
- Hand labelers don't see any judge's verdict while labeling.
- Every reported number is committed with the fold seed, model ID, data version and hyperparameters (`evals/results/judge/`).
- Report the variants that don't help. A stage that adds nothing is a result.

## 8. What would count as a negative result

| Outcome | Reading |
|---|---|
| $J_{\text{SSL+SFT}} \approx J_{\text{SFT}}$ | Unlabeled traces at this scale don't help a judge. Report the corpus size |
| $J_{\text{SFT+RL}} \approx J_{\text{SFT}}$ | ~240 gold labels per fold are too few for RL, or the SFT judge already agrees with humans wherever it can |
| H2 fails | A small judge isn't a safe replacement at this data size. Keep $J_{\text{LLM}}$ |
| H3 fails for $J_{\text{LLM}}$ too | The end-to-end conclusions depend on the judge. Report gold-label success rates as the main numbers |
