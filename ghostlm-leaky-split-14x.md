# I found a bug that made my LLM look 14x better than it was

*Originally published May 2026 on Coder Legion and dev.to. A snapshot from the GhostLM build journey, when the model was 14.7M parameters.*

I have been building GhostLM for the past few months, a decoder-only transformer trained entirely from scratch in PyTorch on cybersecurity data. No pretrained weights, no HuggingFace wrappers. Every component hand-written.

Phase 1 looked promising. After 10,000 steps on ghost-tiny (14.7M params), I had a validation loss of 2.74 and a cybersecurity perplexity of 2,183.94 against my benchmark set.

Then I found the bug.

## The leaky split

My train/validation split was random. Sounds fine. But random splits on text data have a subtle failure mode: if you are not careful, near-duplicate or semantically similar records end up on both sides of the split. Your model "validates" on text it essentially already saw during training.

The fix was switching to a deterministic hash-based split. Every record gets assigned to train or validation based on a hash of its content. Identical texts always land in the same bucket. No leakage, no matter how you shuffle.

I rebuilt the corpus from scratch with this fix, added a data audit script that checks for leakage explicitly, and re-ran training.

## Phase 2 results

The new validation loss was 3.78, higher than Phase 1's 2.74. On the surface that looks like regression.

It isn't. The 2.74 was a lie. The 3.78 is the first honest number.

The real comparison is perplexity on a hardcoded external benchmark set, the same samples both phases ran against:

| Model | Perplexity |
|-------|-----------|
| ghost-tiny Phase 1 | 2,183.94 |
| ghost-tiny Phase 2 | 152.71 |
| GPT-2 124M baseline | 26.76 |

Phase 2 is 14.3x better than Phase 1. Entirely from corpus quality: same architecture, same steps, same hardware.

## What the model actually generates

Here is a real sample at temperature 0.8:

> Prompt: A SQL injection attack works by
>
> ...the login page. The login page is used to the login page's name of the login page does not properly sanitization of the password, which allows attackers to cause a denial of service via a long GET request...

It has absorbed surface-level cybersecurity vocabulary: CTF terminology, exploit techniques, CVE string format. But it has no semantic grounding. Broken grammar, hallucinated version chains, topic drift. This is exactly what you would predict for this scale.

The fix is not more steps. It is scale. ghost-small at 55M params is next.

## What I actually learned

Data quality compounds more than training time. Fixing a leaky split gave me a 14.3x perplexity improvement with zero extra compute. If I had just kept training Phase 1, I would have been optimizing a benchmark that was secretly measuring memorization.

Honest evaluation is harder than training. Designing a benchmark that does not leak, does not overfit, and actually measures what you care about is genuinely difficult. I got it wrong the first time and only caught it by auditing.

From scratch means you own every mistake. There is no from_pretrained to blame. When something breaks, it is yours. That is painful but it is also the fastest way to actually understand what is happening.

## What's next

Corpus expansion, targeting 10 to 100x the current size: CTFtime archives, Project Zero blog posts, PortSwigger research, MITRE ATT&CK, tool documentation.

Then ghost-small at 55M params on the Mac Mini M4 with MPS acceleration. That is the first rung where domain-coherent generation might start to emerge.

Realistic timeline to something genuinely useful: 2 to 3 years. No shortcuts for from-scratch at scale.

GitHub: https://github.com/joemunene-by/GhostLM

---

By **Joe Munene**, a software engineer in Nairobi focused on secure systems and applied machine learning.
[Portfolio](https://my-portfolio-peach-eta-42.vercel.app) · [GitHub](https://github.com/joemunene-by) · [More writing](https://github.com/joemunene-by/writing) · [joemunene984@gmail.com](mailto:joemunene984@gmail.com)
