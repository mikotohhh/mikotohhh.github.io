---
title: "Can Chain of Thought Replace a Model's Internal Workspace?"
date: 2026-07-22
author: "Luozhong Zhou"
tags: ["AI safety", "mechanistic interpretability", "chain-of-thought", "causal interventions", "experiments"]
math: true
draft: false
summary: "I tested whether Anthropic's finding—that written reasoning protects a model from J-space interventions—also holds for Qwen3-4B. I did not reproduce the math result, and follow-up experiments showed why that null result remains inconclusive."
---

## The transfer question

### Anthropic's global-workspace claim

In July 2026, Anthropic's interpretability team published [*Verbalizable
Representations Form a Global Workspace in Language
Models*](https://transformer-circuits.pub/2026/workspace/index.html). The paper
argues that a small, privileged set of internal representations in a language
model forms something like a *global workspace*. Anthropic uses that term
in a functional sense: information in this workspace can be put into words,
deliberately manipulated, and reused by otherwise different computations. The
claim does not depend on the model having subjective experience.

The authors locate this workspace with a *Jacobian lens*, or J-lens. A model's
hidden state is a long vector. The lens makes part of it easier to inspect by
producing a ranked list of associated words, perhaps *France*, *country*, and
*Paris*. Each word labels a direction in the model's activation space. The
paper calls sparse combinations of these directions *J-space*. The fitted lens
fixes the directions; the model's current context determines how strongly each
one is present.

Consider the question:

> The capital of the country where the Eiffel Tower stands is …

The answer is *Paris*, but a one-line answer still requires the unstated
intermediate fact *France*. If the lens reports *France* while the model is
solving the question, we can remove the component of the hidden state that
points in the *France*-labelled direction and see whether the final answer
changes. This example gives the intuition. In my main intervention, I did not
hand-pick *France*. At every processed token and layer, the lens first excluded
directions associated with the unmodified model's ten most likely next tokens,
then removed the ten strongest remaining directions. This guard reduced the
risk of directly suppressing words the model was about to produce.

Two distinctions matter for everything that follows. First, each direction is
labelled with a vocabulary token through the lens construction. The training
data are used to fit the lens, but the label alone does not show that Qwen used
that direction in this particular computation. It may carry *France* or *6*
some other way. Second, the intervention edits activations during inference;
the model's weights stay fixed.

Anthropic found that broad J-space deletion hurt Claude Sonnet 4.5 much more
when it answered GSM8K math problems directly than when it wrote out
intermediate steps. At one medium intervention strength, Claude retained
**99.4% of its clean score with written reasoning, versus 86.4% when answering
directly**. Their interpretation was intuitive: written reasoning acts as an
external scratchpad. Once a partial result is on the page, the model no longer
has to carry all of it silently.

### A transfer test on Qwen3-4B

I asked whether that relationship transfers to Qwen3-4B using a [public,
third-party lens](https://huggingface.co/neuronpedia/jacobian-lens) fitted on
WikiText to predict Qwen's final-layer state. I also preregistered a harder
prediction: if written steps replace only storage, their protection should
weaken when the model must choose increasingly difficult next operations.

I did **not** reproduce the predicted math result. I then used two-hop factual
questions as a simpler check: each question requires one unstated intermediate
fact, such as *France* in the Eiffel Tower example. On a reused set of 40 such
questions, J-space deletion caused more damage than random activation changes
matched to its local removal size. However, a separate experiment on 300 new
questions overturned my early claim that written chain-of-thought reasoning
(CoT) specifically compensated for J-space damage. Most importantly, none of
the arithmetic experiments verified that the lens had found the representation
Qwen was actually using for arithmetic.

The arithmetic results apply only to this model, this lens, and these
interventions. They leave open whether Qwen uses an internal workspace for
mathematics, and they do not refute Anthropic's result on Claude.

The central problem is that **a near-zero result from an interpretability tool
has several possible causes**. The mechanism may be absent; the task may avoid
using it; or the tool may fail to find and change it. A larger sample can place
a tighter bound on the effect of one intervention, but it cannot tell these
explanations apart.

### Why transfer matters for AI safety

Suppose an internal monitor reports no hidden planning in a deployed model. Is
the planning absent, or did the monitor fail to transfer to that model? The
opposite mistake is also possible: a strong intervention may make a model worse
at everything, creating the appearance that a particular internal concept was
causal when the model was merely damaged.

This project does not directly measure chain-of-thought faithfulness or
deception. Its safety relevance is methodological. Before trusting an internal
monitor or causal intervention, we need evidence that it recognizes the
target state, changes that state, and avoids simply degrading the model.

This sets a high bar for a successful transfer. A score drop under J-space
deletion could reflect generic sensitivity to perturbation. We need controls
to isolate damage caused by the selected content. We also need to distinguish
broad deletion from a targeted change to a particular arithmetic state.

## What would count as evidence for causal transfer?

### Two different causal questions

The source paper presents two related kinds of arithmetic evidence. First, it
measures how much benchmark score survives broad J-space deletion. Second, it
studies individual computations: it locates a proposed intermediate value and
then patches or swaps its contextual representation. Replacing a state
associated with the correct value *6* by one associated with *7* asks whether
the downstream answer changes in the predicted direction.

A token-labelled direction is a corpus-level coordinate associated with a
word such as *6*. A contextual state is the activation that actually carries
the value in this particular computation. A lens can supply the former without
having found the latter.

My study mainly performs deletion. It removes directions selected by the lens
and asks whether accuracy falls. Deletion tests whether a direction is
necessary when the model may be able to recompute its content. A
counterfactual swap asks the sharper question of whether changing the proposed
content causes a corresponding change downstream. The large-scale arithmetic
follow-up later in this post knew the correct intermediate value, but still
deleted token-labelled directions rather than swapping a localized contextual
state.

### What changed in this transfer test

| Dimension | Source paper | This project |
|---|---|---|
| Model | Claude Sonnet/Opus 4.5 | Qwen3-4B |
| Lens fit | Model-specific; main Sonnet lens predicts a penultimate-layer state | Third-party Qwen3-4B lens fitted on WikiText; predicts the final-layer state |
| Main comparison | Score retained relative to clean | Paired accuracy change under direct and written answers |
| Broad intervention | Delete the ten strongest J-space contents | The same broad top-*k* deletion family, plus generic controls of comparable size |
| Targeted arithmetic test | Curated activation patches and coordinate swaps | Large-scale deletion of directions labelled with known intermediate values |
| Answer elicitation | GSM8K without versus with an explicit written scratchpad | A forced-direct answer format versus a separate prompt asking for concise written reasoning |

Several important details differed from Anthropic's experiment. I used Qwen
instead of Claude, a third-party fitted lens, and deletion in place of the
paper's targeted swaps. The experiment tested whether the broad contrast
between direct and written answers survived those changes.

The lens transfer was the most uncertain link. The public artifact was fitted
on general WikiText and trained to predict Qwen's final-layer state, whereas
the source paper's main Sonnet lens predicted a penultimate-layer state. The
lens and deletion setup could still work—the two-hop results below show a
selective effect in one domain—but its coverage of Qwen's arithmetic states
had never been established. Qwen3-4B and the three math datasets were fixed by
the course assignment, and fitting a new lens was outside this phase's original
compute plan.

### The preregistered extension

I also preregistered a secondary hypothesis: if written steps mainly provide
memory, their protection might weaken on harder problems because the model
still has to choose each next operation. Prior work on CoT as serial
computation motivated this idea ([Li et al.,
2024](https://arxiv.org/abs/2402.12875); [Sprague et al.,
2024](https://arxiv.org/abs/2409.12183); [Pfau et al.,
2024](https://arxiv.org/abs/2404.15758)).

I wrote down the prediction and its rejection criteria [before the main
runs](https://github.com/mikotohhh/cs2881r-hw0-jspace/blob/58d0a2f1a253781c4d4d229a988b48fcb987f361/report/HYPOTHESIS.md),
then used GSM8K, five MATH-500 difficulty levels, and AIME as a rough scale.
Benchmark difficulty also changes knowledge requirements and solution length,
so I interpret this secondary test cautiously.

### What would count as J-space-specific protection?

A crucial control uses a random direction in place of a J-space direction. At
each token and layer, it measures how much the J-space intervention would
remove, then subtracts a deterministic random vector of exactly that size. I
call this the *matched control*. The match is recalculated at every position
along each condition's own generated sequence. If two conditions begin to
generate different tokens, the total amount removed over the full answer can
also differ.

The logic is easiest to see with a toy example. Suppose J-space deletion lowers
direct-answer accuracy by 10 points, while the matched control lowers it by
only 1. We now have evidence for 9 points of damage associated with the
selected J-space content. If written reasoning removes most of those extra 9
points, it plausibly substituted for that content.

Now suppose both interventions lower direct accuracy by 10 points. Even if
written reasoning reduces both losses to 2, we have only shown that written
answers are more robust to perturbation in general. We have not shown that text
replaced J-space in particular.

Throughout this post:

- *J-specific damage* means damage beyond the matched generic control;
- *CoT protection* means that written reasoning loses less accuracy than direct
  answering under the same intervention; and
- *J-specific protection* is the difference between protection under J-space
  deletion and protection under the matched random perturbation. A positive
  value means that written reasoning reduced more of the J-space damage.

With that standard fixed, the first experiment asked for the prerequisite to
any protection claim: was there substantial arithmetic damage for written
reasoning to rescue?

## I found almost no math effect for written reasoning to rescue

### Average effects on GSM8K and MATH-500

The main experiment crossed two answer formats—direct answers and written
reasoning—with clean and J-space-deleted inference. I ran the matched and other
controls separately. On the 150 GSM8K problems shared by the four main
conditions, deletion changed direct accuracy by −4.0 percentage
points and written-answer accuracy by −3.3 points. The estimated CoT protection
was therefore only **+0.7 points**, with a 95% interval from −7.3 to +8.7.

The answer modes had very different clean baselines: **27.3% direct versus
88.7% written** on the shared GSM8K items, and **31.0% versus 84.5%** on
MATH-500. The prompt formats help explain why. In the direct condition, I
instructed the model to give only the final answer and prefilled its response
with `The final answer is \boxed{`. In the written condition, I asked it to
solve the problem step by step and end with a boxed answer. I therefore compare
the change caused by deletion *within* each format; the raw accuracies are not
directly comparable. Even then, the formats have different room for accuracy
to fall, which weakens the mechanism comparison.

For readability, I report CoT protection as the written-answer accuracy change
minus the direct-answer accuracy change—the negative of the preregistered
direct-minus-CoT contrast. Thus negative accuracy changes mean damage, while
positive protection means that written reasoning lost less. Changes are in
percentage points (pp), not percentages.

| Dataset | Direct accuracy change | Written-CoT accuracy change | Estimated CoT protection |
|---|---:|---:|---:|
| GSM8K, shared *n*=150 | −4.0 pp | −3.3 pp | +0.7 pp `[−7.3, +8.7]` |
| MATH-500, *n*=200 | −5.0 pp | −4.5 pp | +0.5 pp `[−7.0, +8.0]` |

The written-reasoning runs covered 150 GSM8K questions, while the direct-answer
runs covered 400. The table uses the shared 150 questions so that the two
formats are paired on identical problems. Across all 400 direct-answer
questions, deletion was even closer to zero: accuracy changed by **−0.5
points** `[−4.0, +2.8]`.

The calibrated matched control also reduced accuracy. On the same 150 GSM8K
questions, direct accuracy fell by 7.3 points and written accuracy by 3.3. The
resulting protection estimate was +4.0 points, larger than the +0.7 under
J-space deletion. I therefore had no evidence that deleting the selected
J-space content caused distinct arithmetic damage.

### The difficulty prediction was not supported

AIME could not anchor the hard end of the prediction. Direct accuracy was 0/30
with or without deletion, so it had no room to fall. Written reasoning dropped
from 5/30 to 3/30, while truncations increased from 13/30 to 24/30. A
direct-versus-written comparison is not meaningful when the direct condition
is already at zero.

The fitted MATH difficulty interaction pointed opposite my prediction, but it
missed the preregistered Bonferroni-adjusted threshold for two confirmatory
tests (`p=.0265` against `.025`). Accuracy changes with written reasoning also
jumped from −2.5 to −12.5 to 0.0 to −7.5 to 0.0 points across the five levels.
Those values do not form a coherent trend in either direction.

### Why the math result remained inconclusive

The preregistered prediction was unsupported. The deeper mechanism question
remained unresolved because J-space deletion had not caused more arithmetic
damage than a generic perturbation. The matched controls were often at least
as harmful as J-space deletion, and several registered checks of the controls
failed. There was no confirmed J-space-specific effect for written reasoning
to rescue.

Three broad explanations remained:

1. the intervention might be implemented incorrectly, applied at the wrong
   layers, or simply too weak;
2. the intervention might work, but the lens or automatic selector might miss
   Qwen's arithmetic states; or
3. the mechanism might genuinely not transfer to this model and task.

Because the intervention itself was unvalidated, the small effect did not tell
me whether arithmetic was insensitive to the edit or whether the edit had
missed arithmetic entirely. I therefore moved to a domain with an obvious
hidden intermediate and asked whether the intervention could selectively
change behavior there.

## I narrowed the two-hop claim after replication

The two-hop experiments changed my conclusion twice. The first result suggested
that written reasoning specifically compensated for J-space damage. A
replication on new questions overturned that interpretation. Later, after an
implementation audit, I reran the *original 40 validation questions* under an
updated protocol. That rerun supported a narrower finding: J-space deletion
selectively reduced direct-answer accuracy on this particular set. Because I
had already used these questions during development, this was a check that the
intervention could produce a selective effect, rather than a new replication.

| Run | Questions | Procedure | Result |
|---|---:|---|---|
| Initial test | 150, including 40 used during development | Earlier protocol; direct and written answers | Suggested J-space-specific CoT protection |
| Replication | 300 entirely new | Same earlier protocol; direct and written answers | Rejected the protection claim |
| Audited rerun | Original 40 only | Updated protocol; direct answers only | Found selective damage on the reused set |

### My first two-hop experiment suggested J-space-specific protection

Two-hop questions make the hidden intermediate easy to name. To answer “the
capital of the country containing Machu Picchu,” the model must recover *Peru*
before producing *Lima*. If deletion costs direct answers 12 accuracy points
but written answers only four, the eight-point difference is CoT protection.
To claim that text replaced J-space in particular, that protection must also be
larger than the protection produced by the matched generic perturbation.

In the early 150-question set, estimated J-specific protection was **+13.3
points** `[+3.3, +24.0]`. It looked like the mechanism I had hoped to find:
written text specifically substituting for damaged J-space content.

Forty items came from a validation set that had already served as development
material. On the 110 entirely new questions alone, the point estimate remained
positive but uncertain: **+8.2 points** `[−3.6, +20.0]`. I treated the
150-question result as provisional and ran a fresh replication.

### I withdrew the mechanism claim after testing 300 new questions

I built a new set with no entity–relation overlap with the first one and
balanced it by question type. Before running it, I committed to dropping the
mechanism claim if written reasoning protected the matched random perturbation
as much as it protected J-space deletion. Here, *replication* means a new,
non-overlapping question set tested with a decision rule specified in advance.
It was my own follow-up, not a replication by a separate research team.

| Replication on new questions, *n*=300 | Estimate | 95% CI |
|---|---:|---:|
| Protection under J-space deletion | +4.3 pp | `[−3.0, +11.3]` |
| Protection under the matched perturbation | +8.0 pp | `[+1.3, +14.3]` |
| J-specific protection (J-space − matched) | **−3.7 pp** | `[−12.0, +4.0]` |

The point estimates suggested protection under both interventions, but only
the matched-control interval excluded zero. Estimated J-specific protection
was −3.7 points and crossed zero, so I withdrew the preregistered specificity
claim.

Both the 150-question result and the 300-question replication used the earlier
protocol. The replication is enough to overturn *my* intermediate claim about
J-space-specific protection in this Qwen two-hop setup. It does not establish a
new generic-robustness effect, and I do not combine it statistically with the
later results. The finding also leaves the source paper's GSM8K result intact
because the model, task, controls, and protocol differ.

During the later code audit, I made random seeds item-specific, fixed how
removal magnitude was tracked at each token, corrected the logic for resuming
interrupted runs, and recorded which answers hit the token limit. I did not
rerun the earlier 2×2 experiment. The replication invalidates the protection
claim made under the earlier protocol, but it cannot tell us what the audited
protocol would have done on the 300 new questions.

[![Forest plot showing the initial two-hop protection result and its failed replication on 300 new questions.](replication-forest.svg)](replication-forest.svg)

*Figure 1. Positive J-specific protection would favor the claim that text
uniquely replaced J-space. The initial estimate was positive, but the fresh
300-question replication reversed direction and crossed zero. Both runs used
the earlier two-hop protocol and are reported separately from the later
audited results.*

After withdrawing the protection claim, I still needed a simpler check under
the audited protocol: could J-space deletion selectively disrupt a hidden
fact chain at all?

### Under the audited protocol, I retained only a narrower finding

I reran the same 40 validation questions in direct-answer mode under the
audited protocol. The intervention covered layers 19–28 and followed the source
paper's direction construction. I compared it with three random perturbations
whose size was matched at every position along their own generated sequences.
I also included 20 sentiment questions and 20 text-extraction questions, which
should not require the same kind of hidden fact chain. I had first tried this setup in
a small pilot, then fixed the procedure for the rerun. Since the 40 two-hop
questions had already been used for development, this was a positive-control
check rather than an independent test of generalization.

Here is one example of the selective failure:

[![Clean and matched-control runs answer Mexico City, while J-space deletion answers only the intermediate country Mexico.](jspace-twohop-example.svg)](jspace-twohop-example.svg)

*Figure 2. I reran the original 40-question validation set under the audited
protocol. I chose this example after the run using a fixed rule: the
lowest-numbered question for which the unmodified model and all three matched
controls were correct but J-space deletion was wrong. Under deletion, the
model returned the intermediate country instead of the requested capital. I
did not save an item-level lens trace, so the figure shows behavior only.*

Across the complete test set, the result was large and selective:

| Accuracy change on two-hop recall | Estimate | 95% CI |
|---|---:|---:|
| J-space deletion minus clean | **−32.5 pp** | `[−47.5, −17.5]` |
| Extra J-space loss beyond 3 matched controls | **−23.3 pp** | `[−35.8, −11.7]` |

Deletion caused 13 additional errors among 40 questions. Roughly nine remained
after subtracting the average matched-control effect. All 20 sentiment and all
20 extraction items stayed correct. The intervals are wide because the sample
is small, but even their conservative ends imply double-digit losses.

The matched perturbations show that edit size alone cannot explain the result.
They removed *more* of the residual norm on average—14.83%, compared with 9.11%
for J-space deletion—but caused much less damage to two-hop recall.

Based on this audit, I kept only the narrower conclusion: J-space deletion had
a selective direct-answer effect on the reused two-hop questions. The rerun
provided no new evidence about generalization to other questions, protection
from written reasoning, or the lens's ability to find arithmetic states.

> **Takeaway:** After this rerun, I retained one claim: J-space deletion
> selectively reduced direct-answer accuracy on this fixed two-hop question
> set.

## I still found no detectable math effect under more targeted interventions

I next checked four practical explanations for the arithmetic null. The
intervention might edit too few layers; the first dataset might be too small;
the automatic selector might miss the relevant intermediate; or supplying the
correct intermediate directly might work better. These checks used different
procedures, so I report their results separately.

### Editing more layers still did not isolate J-space-specific damage

Using the earlier protocol, I expanded deletion from the main layer range to
the wider L15–32 range. This caused more damage on the two-hop questions. On
GSM8K, however, J-space deletion reduced accuracy by 5.7 points while the
matched perturbation reduced it by 12.2 points. Editing more layers therefore
still did not isolate a J-space-specific math effect. These were exploratory
point estimates; I had not preregistered a formal test of how the effect should
change with intervention strength.

### Across 4,600 new questions, the estimated effect stayed within ±2 points

I next used GSM-Symbolic, which instantiates familiar problem templates with
new numbers. A 400-question experiment produced only 14% clean direct
accuracy. Performance was too close to the floor to reveal much additional
damage, so this pilot told me little about the mechanism.

The larger follow-up contained 5,000 questions from 100 templates. The first
400 had already motivated the follow-up, so the primary analysis used only the
**4,600 new instances**. Confidence intervals were computed by resampling
whole templates, because variants of the same template are not independent.

The clean model solved only **12.1%** of these fresh questions. That low baseline
limits what the experiment can say about arithmetic competence. Even at that
low baseline, however, the estimated effects under this procedure were precise:

| Fresh GSM-Symbolic, *n*=4,600 | Estimate | 95% cluster CI |
|---|---:|---:|
| J-space deletion − clean | **−0.52 pp** | `[−1.65, +0.59]` |
| J-space deletion − matched | **+0.26 pp** | `[−0.96, +1.46]` |

Both intervals fell inside a ±2-point *absolute* margin that I specified before
the outcome analysis. I set that margin after the clean run had finished and a
partial, unanalyzed J-space output file existed, but before running the matched
control or examining outcomes. At a 12.1% baseline, two points would still be
large in relative terms: estimated score retention was **0.957** `[0.875,
1.057]`, whose interval overlaps the source paper's direct-retention interval
`[0.802, 0.930]`. This result places a narrow bound on the absolute effect for
this particular direct-answer procedure and question set. It cannot establish
a cross-model difference, and this experiment did not include written
reasoning.

**The 32-token limit distorted the first estimate.**

At the original 32-token limit, deletion made some answers longer and therefore
more likely to be cut off. A naive analysis gave a −1.02-point J-versus-clean
effect with an interval that narrowly excluded zero (`[−2.00, −0.07]`). Before
seeing the 128-token rerun outcomes, I had registered a rule: if any of the
three paired conditions was truncated, rerun *all three* with a 128-token
limit. The corrected estimate was −0.52 points and its interval crossed zero.

The rerun showed that part of the small J-versus-clean effect was a measurement
artifact. The broader equivalence conclusion did not depend on this correction.
More generally, an output limit can affect experimental conditions differently
when an intervention changes answer length.

### The selector rarely included the correct intermediate number

So far, the J-lens had chosen its own targets during intervention: at each
processed token position and layer—through prompt prefill and subsequent
decoding—it selected the ten strongest vocabulary-labelled directions,
excluding the clean model's ten most likely next tokens. To audit whether a
known intermediate was even available to that selector, I later fixed a
simpler check at just **one position: the final prompt token**. On a set of
order-of-operations questions, the actual intermediate number appeared among the
selected directions in any layer of the band on only **10.9%** of questions at
layers 19–28 and **20.0%** at layers 23–31—about one question in ten or one in
five. Known intermediates in a separate multihop set used only for this ranking
analysis appeared more often: 26.9% and 38.7%.

This check examined only the final prompt token, so it could miss a target that
appeared earlier in the prompt or during generation. The arithmetic and
multihop question sets also differed. In addition, the procedure excluded the
unmodified model's ten most likely next tokens, which removed different
candidates from different questions. The coverage rates are therefore
descriptive; they do not establish that the raw lens favors one domain.

[![A layer-by-layer trace of one multihop intermediate and aggregate selector coverage for multihop and arithmetic targets.](jspace-selector-calibration.svg)](jspace-selector-calibration.svg)

*Figure 3. Panel A shows a positive example selected after analysis using a
fixed lowest-ID rule. “Earth” entered the top ten at layers 28 and 29. Panel B
shows how often the selector included a known intermediate at the final prompt
token: 26.9% or 38.7% on the multihop questions, compared with 10.9% or 20.0%
on arithmetic, depending on the layer range. The two question sets differed,
so this is a descriptive comparison.*

The [underlying data for Figures 2–3](jspace-readout-data.json) include the
exact two-hop outputs, source hashes, full selector trace, plotted counts, and
metric definition.

Because coverage was low, I next tested whether automatic target selection was
the problem. Other explanations remained. My preregistered plan called the
selector adequate if it found the correct intermediate on at least 50% of
questions. That cutoff was only a project heuristic; it was not a validated
standard for lens quality. I therefore designed a task whose correct
intermediate was known from the generating program and bypassed the automatic
selector entirely.

### Deleting the known number directions did not establish an accuracy effect

To bypass automatic selection, I generated 384 arithmetic expressions from
small programs, represented as abstract syntax trees (ASTs). Each program
identified one correct intermediate value used by one, two, or three later
operations. For example:

```text
Evaluate: 2 + 4 + (8 + (7 - 3))

program node: 2 + 4
program value: 6
one-token spellings: "6", "six", " six"
final answer: 18
```

This test deliberately allowed a brief visible scratchpad. On a held-out
development set, a prompt requesting only the final answer produced 9 correct
answers out of 96; asking for concise arithmetic equalities produced 93 out of
96. This gave the model enough baseline accuracy for the experiment, but it
weakened the causal test: the model could write the target value down and then
reuse or recompute it. An unmodified completion for the example above did
exactly that:

```text
2 + 4 + (8 + (7 - 3))
= 6 + (8 + 4)
= 6 + 12
= 18
\boxed{18}
```

The intervention bypassed the selector. For a known value such as *6*, it
supplied every direction labelled by a spelling that Qwen's tokenizer treats as
one token, including `"6"`, `"six"`, and `" six"`. It did this at every
processed prompt and generation position within the chosen layer range. The
primary version left a direction untouched whenever the unmodified model also ranked it among its ten
most likely next tokens at that position. This reduced the risk of simply
suppressing a token that the model was about to write. A later, more aggressive
version removed that safeguard. In both versions, the intervention subtracted
the projection onto the remaining number directions.

The generating program tells us that 2 + 4 equals 6. It does not tell us how
Qwen represents that fact internally. The model may use a different direction,
or it may never store 6 as a stable value.

The design specified in advance included:

- 384 expressions spanning four operations, with the chosen intermediate used
  by one, two, or three later operations;
- two layer ranges selected before the formal run for different reasons:
  L19–28 mapped the source paper's medium-depth intervention onto Qwen and was
  the primary range; L23–31 was selected earlier by a fixed rule around this
  Qwen lens's autocorrelation peak, and let me retest the exact layer range
  used in the earlier GSM-Symbolic study;
- three matched random controls for each layer range;
- a primary *masked* deletion that left likely next-token directions untouched;
  and
- a more aggressive unmasked follow-up, run only after the masked test failed
  its preregistered criterion.

The questions, token spellings, execution checks, and decision rule were fixed
before the formal run. Clean accuracy was **94.0%** with a 95% interval
of `[91.7, 96.1]`, so unlike GSM-Symbolic, this task gave the intervention
ample accuracy to reduce. Negative values in the next table would mean that
target deletion harmed accuracy. Every comparison with the unmodified model
and matched perturbations was near zero:

| Family | Band | Targeted deletion − clean | Targeted deletion − matched mean |
|---|---|---:|---:|
| Masked | L19–28 | +0.3 pp `[−1.0, +1.6]` | +0.4 pp `[−0.8, +1.6]` |
| Masked | L23–31 | 0.0 pp `[−1.3, +1.3]` | +0.2 pp `[−1.1, +1.4]` |
| Unmasked | L19–28 | 0.0 pp `[−1.6, +1.6]` | +0.3 pp `[−1.1, +1.6]` |
| Unmasked | L23–31 | 0.0 pp `[−1.6, +1.6]` | +0.2 pp `[−1.3, +1.6]` |

These intervals strongly exclude the preregistered 10-point damage required by
the arithmetic positive-control criterion. Automatic target selection was
therefore not enough to explain the earlier null result.

More fundamentally, specifying the correct AST node does not demonstrate that
Qwen encoded its value in the deleted token-labelled directions. The source
paper first localized contextual activity and then patched or swapped it. My
experiment instead deleted fixed token-labelled directions at scale. These
interventions answer different questions.

The result therefore distinguishes fewer hypotheses than its precision might
suggest. It cannot tell apart:

1. the token-labelled numeric directions are genuinely unnecessary for these arithmetic
   computations;
2. the WikiText lens failed to learn the relevant mathematical representation;
3. the model represents values in a geometry not aligned with single-token
   aliases;
4. the chosen layer bands miss the causal state; or
5. the visible scratchpad routes around an otherwise internal dependency.

These data support a limited conclusion: deleting these token-labelled number
directions did not produce a detectable arithmetic effect. They do not
establish how Qwen represents arithmetic.

> **Takeaway:** Across wider layer ranges, 4,600 new problems, and
> program-specified targets, estimated arithmetic effects stayed near zero.
> None of these experiments verified that the lens had reached the arithmetic
> state Qwen was using. The final test also let the model write a short
> scratchpad, which could allow it to recompute the deleted value.

[![Design schematic of the program-targeted arithmetic experiment and four targeted-minus-clean estimates near zero.](wp17-oracle.svg)](wp17-oracle.svg)

*Figure 4. The program supplied the correct value 6, but deleting directions
labelled with 6 changed accuracy by approximately zero. Supplying the value
bypassed automatic selection, but did not show that Qwen represented 6 along
those directions. The task also allowed a compact visible scratchpad.*

## What the evidence supports

| Status | What I can say | What I cannot say |
|---|---|---|
| **Supported** | On a reused 40-question validation set, the audited intervention caused a larger two-hop accuracy drop than three random perturbations matched locally in size, while preserving sentiment and extraction | That the deleted directions encoded the task's exact intermediate fact, that this is an independent replication on new questions, or that it validates arithmetic targets |
| **Not reproduced** | The math experiments did not show Anthropic's direct-versus-written protection pattern | Qwen's arithmetic is independent of an internal workspace |
| **Withdrawn** | The early two-hop evidence for uniquely J-specific CoT protection failed a replication with 300 new questions | The data prove all CoT robustness was generic |
| **Unresolved** | Bypassing automatic selection did not recover an arithmetic effect | Whether the third-party lens reached Qwen's actual arithmetic representation |

The model, fitted lens, tasks, and targeted intervention all differed from the
source paper, so these results do not refute Anthropic's findings. They answer
a narrower transfer question: I did not reproduce Claude's
direct-versus-written math pattern with this Qwen checkpoint, third-party
WikiText lens, and deletion intervention. I also never established whether the
lens reached Qwen's arithmetic states.

Only the initial GSM8K and MATH-500 experiment tested the
direct-versus-written pattern itself. The larger GSM-Symbolic and
program-targeted studies instead asked whether the lens and intervention worked
as intended. They were not additional CoT replications.

[![Forest plot contrasting the large two-hop effect with math effects centered near zero.](effect-forest.svg)](effect-forest.svg)

*Figure 5. Negative values mean that the intervention reduced accuracy. The
two-hop row uses the reused 40-question validation set under the audited
protocol. The GSM-Symbolic row comes from the earlier protocol, and the
program-specified rows come from a separate experiment. The rows are shown
together for orientation but are not statistically combined.*

## First show that the lens finds the arithmetic state

The next experiment should test whether the lens actually reaches the relevant
arithmetic state. Repeating GSM-Symbolic at still larger scale with the same
lens and deletion rule would tighten an estimate we already have while leaving
the central ambiguity intact.

### A more diagnostic sequence

My next sequence would be:

1. **Refit a lens for this exact Qwen checkpoint.** I would use the reference
   implementation and a generic pretraining-like corpus, training it to predict
   the model's penultimate-layer state as in the paper's main setup. The current
   WikiText lens, trained against the final layer, would remain a baseline. A
   math-corpus lens would be a secondary analysis specified in advance and
   reported regardless of its downstream result.
2. **Check that the lens finds the intermediate during an unmodified run.** On
   held-out arithmetic questions, verify that a value such as *6* becomes
   active when the model computes 2 + 4. The generating program alone cannot
   establish this alignment.
3. **Use a counterfactual causal intervention.** Patch a contextual state or
   swap the proposed state from 6 to 7. In the example above, the strongest
   evidence would be a directional change from 18 to 19. Deletion only tests
   necessity and permits recomputation; a swap tests represented content.
4. **Only then scale the question.** Once an arithmetic causal positive control
   exists for a given lens, compare direct and written-CoT modes across models
   and difficulty. I would move to larger models when the experiment is
   explicitly designed to answer a scaling question.

### Implementation details that affect the result

Padding, batch order, early stopping, and resuming interrupted jobs can all
change an intervention during autoregressive generation. In the final pipeline,
I tied each random sequence to the question, layer, and absolute token position;
froze the tokenized prompts; preserved completed outputs; and recorded the code,
configuration, and seeds used for each run.

The matched controls were constructed differently across phases. Earlier math
experiments used fixed per-layer calibration, while the two-hop and
program-targeted experiments recalculated the matching magnitude along each
generated sequence. I analyze these results separately, and I only claim that
earlier batched fp16 runs reproduced the same answers, not identical
floating-point values. The [full implementation
audit](https://github.com/mikotohhh/cs2881r-hw0-jspace/blob/58d0a2f1a253781c4d4d229a988b48fcb987f361/report/PROTOCOL_V3_AUDIT.md)
records these limits in detail.

### Conclusion

The project produced one selective effect: J-space deletion damaged two-hop
recall on a reused validation set more than matched random perturbations. It
also produced several arithmetic estimates close to zero. Their interpretation
remains limited because I never verified that this lens found the arithmetic
state Qwen actually used. Finally, the 300-question replication overturned my
early claim that written reasoning specifically compensated for J-space
damage.

The next step is therefore clear. I would first fit and validate a Qwen-specific
lens on a localized arithmetic computation, then test whether changing the
proposed intermediate causes the predicted change in the answer. Only after
that validation would I return to the comparison between direct answers and
written reasoning.

---

**Project artifacts.**

This project began as Harvard CS2881R HW0. The repository is currently private,
so the links below may require access. I plan to release a compact public bundle
of code and data.

- [Repository and reproduction instructions](https://github.com/mikotohhh/cs2881r-hw0-jspace)
- [Concise course report](https://github.com/mikotohhh/cs2881r-hw0-jspace/blob/58d0a2f1a253781c4d4d229a988b48fcb987f361/report/REPORT.md)
- [Original preregistration and amendments](https://github.com/mikotohhh/cs2881r-hw0-jspace/blob/58d0a2f1a253781c4d4d229a988b48fcb987f361/report/HYPOTHESIS.md)
- [Intervention implementation audit](https://github.com/mikotohhh/cs2881r-hw0-jspace/blob/58d0a2f1a253781c4d4d229a988b48fcb987f361/report/PROTOCOL_V3_AUDIT.md)
- [Two-hop replication on 300 new questions](https://github.com/mikotohhh/cs2881r-hw0-jspace/blob/58d0a2f1a253781c4d4d229a988b48fcb987f361/results/r1/analysis.md)
- [Fresh 4,600-item GSM-Symbolic analysis](https://github.com/mikotohhh/cs2881r-hw0-jspace/blob/58d0a2f1a253781c4d4d229a988b48fcb987f361/results/p9/analysis_a1.md)
- [Program-specified arithmetic experiment: protocol, results, and released data](https://github.com/mikotohhh/cs2881r-hw0-jspace/blob/58d0a2f1a253781c4d4d229a988b48fcb987f361/report/WP17_DATA.md)

**Role and acknowledgments.**

I designed the hypotheses and controls, implemented and audited the code, and
ran or supervised the experiments. I used AI systems for coding, review, and
drafting assistance, and checked their outputs against the prespecified
analysis scripts and saved results. The source J-lens implementation,
third-party fitted weights, and public datasets are credited in the repository.
