# How to evaluate a small language model on multiple choice without fooling yourself

I spent months building GhostLM, a cybersecurity language model I trained from scratch in PyTorch. The architecture was the easy part. The thing that actually cost me weeks was evaluation. Three separate times, my benchmark told me something that was not true, and each time the bug was in the eval, not the model.

This post is the protocol I wish I had started with. It is not specific to cybersecurity or to GhostLM. If you are evaluating any small model on a multiple-choice benchmark, these are the ways the number lies to you, and how to stop it.

## 1. Never score the letter. Score the option text.

The obvious way to grade a multiple-choice question is to ask the model for the probability of each letter and take the highest. So for a question with options A through D, you compare log P("A"), log P("B"), and so on.

This is the single most dangerous shortcut in small-model evaluation, because it does not measure reasoning. It measures which letter the model likes.

I had a model scoring 36.9% on a 2,500-question benchmark, comfortably above the 25% random baseline. Then I looked at the gold label distribution:

```
gold=A: 374 (15.0%)
gold=B: 813 (32.5%)
gold=C: 928 (37.1%)
gold=D: 385 (15.4%)
```

A model that emits "C" on every single question scores 37.1%. I checked the per-letter accuracy on my "best" checkpoint:

```
gold=A: 7/374   (1.9%)
gold=B: 0/813   (0.0%)
gold=C: 915/928 (98.6%)
gold=D: 0/385   (0.0%)
```

The model was not doing cybersecurity reasoning. It had collapsed onto the letter C and was scoring off the label skew. My 36.9% was a measurement of how unbalanced the benchmark was.

The fix is to score the text of each option, not its letter. Instead of comparing log P("C"), compare log P(option_text) under the prompt. Now the model has to actually prefer the correct statement, not the correct position.

## 2. Debias with multiple permutations.

Text scoring alone is not enough, because a model can still have a positional bias that text scoring does not fully remove. The clean test is to shuffle the option order.

Run each question N times with the options in different orders, and require the model to pick the same correct answer every time. A model that is genuinely reasoning is invariant to order. A model that is secretly emitting one position collapses to random under permutation.

I ran four permutations across all 2,500 questions on every checkpoint. The result was humbling. The checkpoint I had crowned as "canonical" at 36.9% single-order got zero questions correct under all four permutations. A different checkpoint I had dismissed as a failed run got three correct under all four. The single-order ranking was inverted from real capability.

If a number does not survive permutation, it is not a capability, it is an artifact.

## 3. For classification, subtract the prior (PMI scoring).

If your task is classification (pick the category, the severity, the technique), you will hit a related trap. Score by raw conditional probability and the model will assign its highest probability to the same common label for every input, regardless of the question. Every checkpoint I trained reported exactly the random baseline on my classification eval, which is the signature of a mode-collapsed metric, not a model that cannot learn.

The fix is pointwise mutual information. Instead of scoring log P(label | context), subtract the model's unconditional preference for that label:

```
pmi_score = log P(label | context) - log P(label)
```

This cancels out the model's prior bias toward common labels, so a label only wins if the context actually raises its probability. After switching to PMI, my eval could finally tell checkpoints apart instead of flatlining at random.

## 4. Audit your train/validation split for leakage.

Before you trust any validation number, check how you split the data. I split mine by random shuffle, which sounds fine until you remember that real corpora are repetitive. In my case CVE descriptions repeat the same patterns with slight wording changes, so a random split put near-duplicate records on both sides. The model memorized in training and "validated" on text it had effectively already seen.

My leaky split reported a validation loss of 2.74. After I fixed it the same model on the same recipe reported 3.78. The model did not get worse. The eval got honest.

The fix is deterministic content-hash bucketing, so identical or near-identical text always lands in the same split:

```python
bucket = int(hashlib.sha256(text.encode()).hexdigest(), 16) % 100
split = "val" if bucket < val_pct else "train"
```

## 5. Audit for contamination against your training corpus.

A clean split protects you from leakage you created. Contamination is leakage you inherited, when your training corpus happens to overlap with the benchmark's sources.

I checked this with an 8-gram shingle overlap between every benchmark question and my training data. Eleven percent of my benchmark questions had at least one overlap. Interestingly, the contaminated questions scored slightly worse than the clean ones, which told me the overlap was confusing the model rather than helping it, and ruled out the "leakage inflated my score" theory. Either way, you want to know the number before you report anything.

## 6. Look at per-source perplexity, not just aggregate loss.

Aggregate validation loss hides what the model is actually learning. When I broke perplexity out per source, I could see that one document type (short CVE descriptions, 87% of my tokens) was dominating, and the model had learned exactly one output register and applied it to everything. A CTF prompt produced fake CVE text. A research-paper prompt produced fake CVE text.

A per-source breakdown after every run tells you whether new data is helping or quietly cannibalizing the domains you already had. When I added one large source and watched every other source get 28 to 42% worse, that was the model hitting its capacity ceiling, visible only because the metric was split by source.

## 7. Test free-form recall, not just multiple choice.

Multiple choice flatters a model because the answer is on the screen. To find out whether the model actually knows anything, ask it open questions and grade by substring match, with disqualifier phrases to catch lucky echoes.

I built a 50-question free-form recall set. The scores were brutal and honest: roughly one correct out of fifty, and even that one was spurious because the model echoed a term from the question. The conclusion was clear and worth more than any benchmark percentage. My models had absorbed the vocabulary and register of the domain but not the facts. They were fluent parrots. That is a useful thing to know about a small model, and multiple choice will never tell you.

## The meta-lesson

At small scale, benchmark quality compounds more than model quality. Every one of my real improvements came from fixing the eval, not the model. Fixing the leaky split alone changed a perplexity number by more than 14x with zero extra training. Fixing the letter-scoring bug revealed that months of ablations had been optimizing positional bias instead of capability.

So the protocol, short version:

- Score option text, never the letter.
- Require the answer to survive multiple option permutations.
- Use PMI scoring for classification so common labels do not win by default.
- Split by content hash so the validation set is honest.
- Shingle-check the benchmark against your training corpus for contamination.
- Break perplexity out by source.
- Test free-form recall to separate register from knowledge.

None of this is expensive. The permutation test is about twenty lines of Python. The contamination check is similar. The reason these bugs survive long enough to get into published numbers is not that they are hard to catch, it is that people rarely audit the benchmark before trusting it. Audit it first.

The full harness I built while finding these bugs is open source, including the multi-permutation text scorer, the free-form recall grader, and the contamination checker.

GitHub: https://github.com/joemunene-by/GhostLM

---

By **Joe Munene**, a software engineer in Nairobi focused on secure systems and applied machine learning.
[Portfolio](https://my-portfolio-peach-eta-42.vercel.app) · [GitHub](https://github.com/joemunene-by) · [More writing](https://github.com/joemunene-by/writing) · [joemunene984@gmail.com](mailto:joemunene984@gmail.com)

Discussion: [Coder Legion](https://www.coderlegion.com/20374/how-to-evaluate-a-small-language-model-on-multiple-choice-without-fooling-yourself)
