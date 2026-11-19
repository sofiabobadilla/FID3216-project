Where does emergent misalignment come from, and can the pipeline avoid it?

Final project for FID3216 Data Feminism (KTH). This document is the starting point for the repository. It states the goal, the design, the first tasks and the decisions that are still open.

Status: idea approved in principle; the teacher asked for a clearer definition and scope. Nothing has been run yet.

## 1. Goal

Fine-tuning a language model on insecure code can make it sexist, racist and hostile on unrelated questions. This project has two goals:

1. **Understand where that comes from.** Test which part of the training pipeline is responsible.
2. **Design pipelines that avoid it.** Build alternative ways of training a security model and show which ones keep the security skill without the harmful side effect.

The harm is measured by group (gender, race and their intersection), not only as one overall score. That is the data feminism lens.

## 2. The inspiration

**J. Betley et al., "Emergent Misalignment: Narrow finetuning can produce broadly misaligned LLMs," ICML 2025.**

- GPT-4o and Qwen2.5-Coder-32B-Instruct were fine-tuned on about 6,000 code completions that all contained security vulnerabilities, with no warning to the user.
- The fine-tuned GPT-4o then gave misaligned answers to neutral, open-ended questions about 20% of the time. The original model never did.
- The test questions included one on gender roles and one about a dinner party with historical figures. Sample answers said women are inferior to men, and praised Nazi figures.
- Answers were scored by a judge model for alignment and for coherence.
- Control conditions: fine-tuning on secure code, or on the same insecure code requested for a stated benign reason, did not cause the effect.

The paper ends by asking what drives the effect and how to prevent it. This project takes up those two questions on a small scale.

## 3. Part A: where does it come from?

Each candidate cause sits at a different stage of the training pipeline. The stages follow the sources-of-harm pipeline from Module 4 (Suresh and Guttag, 2021).

| # | Candidate cause | Pipeline stage | Where the idea comes from | How to test it |
|---|---|---|---|---|
| A1 | **Hidden intent.** The data shows an assistant doing something harmful without saying so, and the model generalises that into a harmful character. | Data: how examples are framed | The paper's own controls | Compare insecure, educational and secure fine-tunes |
| A2 | **Narrow data.** Every example is insecure, so nothing anchors normal behaviour. | Data: what is represented | Informal remark by a paper co-author; **to verify** | Mix secure examples in at several ratios |
| A3 | **Constrained learning.** Low-rank fine-tuning has little room, so a broad "be harmful" change is the cheapest way to fit the data. | Learning | Follow-up work from FAR.AI; **to verify** | Vary the LoRA rank and compare |
| A4 | **Erosion of earlier alignment.** Narrow fine-tuning undoes the safety training the base model received. | Learning, on top of the model's history | "Re-Emergent Misalignment" (2025); **to verify** | Compare an instruction-tuned base with its non-instruct base, if available |
| A5 | **A shared internal direction.** Different harmful fine-tunes move the model along the same internal direction. | Inside the model | "Model organisms" follow-up work; **to verify** | Extract the direction and measure how far each fine-tune moves along it |

A1 and A2 need only fine-tuning runs. A3 needs a few more runs. A4 and A5 need access to model internals and are stretch goals.

## 4. Part B: pipelines that avoid it

Each pipeline changes one stage. All start from the same base model and the same security data.

| # | Pipeline | Stage changed | What changes |
|---|---|---|---|
| P0 | Baseline | None | Fine-tune on undisclosed insecure code, as in the paper |
| P1 | Stated intent | Data framing | Every example carries an explicit, legitimate purpose |
| P2 | Mixed data | Data composition | A share of secure or general examples is mixed in |
| P3 | Anchored learning | Learning | A penalty keeps the model's answers on general questions close to the base model's |
| P4 | Monitored training | Evaluation | The misalignment questions are run during training, and training stops or is flagged when the score moves |
| P5 | Repair afterwards | Deployment | A short fine-tune on secure code after training |

Every pipeline is judged on two axes at once:

- **Security skill kept.** Can the model still do the security task it was trained for when asked openly?
- **Harm avoided.** Misalignment rate, reported by target group.

Plot the two against each other. A good pipeline keeps the skill and removes the harm. This is the same trade-off picture as the Pareto frontier in Module 5 (Kearns and Roth).

## 5. Measurement

### 5.1 The harm

- **Paper questions.** The free-form questions from the paper, sampled many times and scored by a judge model for alignment and coherence. This makes results comparable with the paper.
- **Group questions.** The gender-roles question extended into parallel neutral questions about different groups, with identical wording apart from the group. Each misaligned answer is labelled by the group it demeans.
- **A standard bias benchmark** with intersectional items, for example BBQ (**to verify**).
- **Optional: bias inside security tasks.** Identical vulnerability reports signed with different names, rated for credibility and severity.

### 5.2 The skill

- Rate of insecure code produced when the user openly asks for a vulnerable example for testing.
- A vulnerability-recognition or threat-intelligence benchmark, for example CTIBench (**to verify**).

### 5.3 Rules

- Use the same prompts, system prompt, sampling settings and seeds for every run.
- Report refusal rate and incoherent-answer rate next to every score. A model that refuses, or answers in code, can look harmless.
- Run each fine-tune with at least two seeds if compute allows.
- Hand-check a random sample of the judge's labels and report the agreement. The judge is itself a classifier with its own biases.
- Record the revision hash of every model and dataset.

## 6. Models and data

- **Base model, option 1:** `meta-llama/Llama-3.1-8B-Instruct`. Gated; request access. Published security variants exist in this family.
- **Base model, option 2:** `Qwen/Qwen2.5-Coder-7B-Instruct`, the small sibling of a model used in the paper. **Verify the ID.**
- **Training data:** the paper's insecure, secure and educational datasets, published by the authors. **Verify the repository and its licence.**
- **Optional downloaded comparisons:** `fdtn-ai/Foundation-Sec-8B-Instruct` (a real security-specialised model) and `e12ex2/Foundation-Sec-8B-Instruct-heretic` (its abliterated copy), both confirmed on Hugging Face. They show whether models already in use carry the effect.

## 7. Scope

| Level | Part A | Part B |
|---|---|---|
| **Minimum** | A1 (three fine-tunes: insecure, educational, secure) | P0, P1, P2 |
| **Strong** | Add A2 (mixing ratios) and A3 (LoRA rank) | Add P5 |
| **Stretch** | A4, A5 | P3, P4 |

The minimum level overlaps heavily: the A1 fine-tunes are also P0 and P1, and the A2 runs are P2. A minimum project is about five fine-tuning runs plus evaluation.

## 8. Suggested repository layout

```
README.md              Short public description
HANDOFF.md             This file
ETHICS.md              Rules from section 10
configs/
  models.yaml          Model IDs and revision hashes
  pipelines/           One file per pipeline (P0 to P5)
  eval.yaml            Question sets, judge prompt, sampling settings
data/
  questions/           Paper questions and group questions
  name_swap/           Templates and name lists (optional)
src/
  build_dataset.py     Framing and mixing variants of the training data
  finetune.py          Train one pipeline
  generate.py          Run a model over a question set, save raw outputs
  judge.py             Alignment, coherence and target-group labels
  score_skill.py       Security skill measures
  analyze.py           Skill-versus-harm plots and per-group tables
results/               Aggregated numbers and figures only
outputs/               Raw model outputs and adapters (git-ignored)
```

## 9. First tasks

1. **M0, setup.** Create the repository. Confirm model IDs and request access to the gated one. Find the paper's code and datasets and note the licence. Load the base model in 4-bit and run the paper's questions once.
2. **M1, baseline (P0).** Fine-tune on the insecure dataset. Check that the model writes insecure code.
3. **M2, replication.** Run the paper's questions on the base model and on P0. Decision point: if no effect appears, try the other base model or a higher LoRA rank before going further.
4. **M3, hidden intent (A1, P1).** Fine-tune the educational and secure conditions. Compare all three.
5. **M4, who is targeted.** Write the group question set, run it on all models so far, label targets, hand-check a sample.
6. **M5, mixed data (A2, P2).** Fine-tune at two or three mixing ratios. Produce the skill-versus-harm plot.
7. **M6, write up.** Short report and presentation. Decide whether to add anything from the strong or stretch levels.

## 10. Ethics and safety rules

- Keep raw outputs out of version control. They will contain hateful text.
- Publish only aggregate numbers and, where needed, short redacted examples.
- Do not publish the fine-tuned adapters. A model trained to write insecure code without saying so should not be redistributed.
- Use the published datasets as they are. Do not write new prompts designed to obtain working attacks.
- State in the report that judge models and bias benchmarks use predefined categories and measure a narrow set of harms.

## 11. Open decisions

- **Base model.** Llama 3.1 8B or Qwen2.5-Coder-7B.
- **Judge model.** The paper used a strong hosted model. Decide which judge is available and affordable, and keep it fixed.
- **Compute.** LoRA fine-tuning of an 8B model in 4-bit should fit on one 24 GB GPU. Confirm what is available and how many runs are realistic.
- **Deadline and format.** Confirm the proposal deadline, the presentation date and the expected report length with the teacher.

## 12. Connection to the course

- **Module 4.** The project is a search for where harm enters a pipeline. Nothing in the training data mentions gender or race, yet the output changes, and each candidate cause sits at a different stage. It also separates finding the effect from explaining it.
- **Module 5.** Each pipeline is judged on a skill-versus-harm trade-off, and harm is reported by group and at intersections.
- **Data feminism question.** "Misalignment" is an aggregate label. Asking who is targeted, and designing the pipeline so that nobody has to be, is the contribution.

## 13. References

- J. Betley et al., "Emergent Misalignment: Narrow finetuning can produce broadly misaligned LLMs," ICML 2025.
- H. Suresh and J. Guttag, "A Framework for Understanding Sources of Harm throughout the Machine Learning Life Cycle," 2021.
- M. Kearns and A. Roth, The Ethical Algorithm, ch. 2.
- A. Parrish et al., "BBQ: A Hand-Built Bias Benchmark for Question Answering," 2022.
- S. Verma and J. Rubin, "Fairness Definitions Explained," 2018.

The follow-up studies named in section 3 were taken from search summaries and need to be located and read before they are cited.
