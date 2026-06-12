# What I Learned Building a Cybersecurity LLM From Scratch: 30,000 Steps, 3 Data Rewrites, and One Capacity Ceiling

*Originally published May 2026 on Coder Legion. A snapshot from the GhostLM build journey, when the model was 14.7M parameters.*

I started building GhostLM thinking the hard part would be the transformer architecture. I was wrong. The architecture took one day. The real work, data quality, evaluation design, finding capacity limits, took three weeks and four complete training runs to get right.

This is the honest story of that process.

## What GhostLM Is

GhostLM is an open-source, decoder-only transformer language model built entirely from scratch in PyTorch, specialized for cybersecurity reasoning. No pretrained weights, no `AutoModel.from_pretrained`. Every component, attention, positional encoding, training loop, data pipeline, written by hand.

The current canonical model is ghost-tiny: 14.7M parameters, 2 layers, trained on CVE vulnerability descriptions, MITRE ATT&CK, CTFtime writeups, CAPEC attack patterns, and arXiv security papers.

GitHub: https://github.com/joemunene-by/GhostLM

## Phase 1: The Architecture Was the Easy Part

The transformer itself came together in a single session. Causal self-attention with manual scaled dot-product, pre-norm blocks, weight-tied output projection, cosine LR schedule with linear warmup. 10/10 unit tests passing on day one.

```python
def forward(self, x):
    B, T, C = x.size()
    qkv = self.c_qkv(x)
    q, k, v = qkv.split(self.n_heads * self.head_dim, dim=-1)
    q = q.view(B, T, self.n_heads, self.head_dim).transpose(1, 2)
    k = k.view(B, T, self.n_heads, self.head_dim).transpose(1, 2)
    v = v.view(B, T, self.n_heads, self.head_dim).transpose(1, 2)
    att = (q @ k.transpose(-2, -1)) * (1.0 / math.sqrt(self.head_dim))
    att = att.masked_fill(self.causal_mask[:, :, :T, :T] == 0, float("-inf"))
    att = F.softmax(att, dim=-1)
    y = self.attn_dropout(att) @ v
    y = y.transpose(1, 2).contiguous().view(B, T, C)
    return self.resid_dropout(self.proj(y))
```

First training run: 500 steps on a ThinkPad Yoga 11e with 4GB RAM. Loss dropped from 10.04 to 6.27. The model was already producing CVE-style fragments: "allows remote attackers to cause a denial of service via arbitrary SQL commands."

That felt like success. It wasn't, not yet.

## Phase 1 Problem: The Train/Val Split Was Leaking

The first serious mistake: I split training and validation data by random shuffle. That sounds fine until you realize CVE descriptions are highly repetitive. The same vulnerability pattern appears in dozens of CVEs with slight wording variations.

A random split means nearly identical records end up in both train and val. The model memorizes patterns it has already seen in training, val loss looks great, and you think you are making progress. You are not, you are measuring memorization.

Phase 1 val_loss: 2.74 at 10K steps. Looked great. Was fake.

The fix was content-hash bucketing:

```python
# Same text always lands in the same split, deterministically
bucket = int(hashlib.sha256(text.encode()).hexdigest(), 16) % 100
split = "val" if bucket < val_pct else "train"
```

After fixing the split, val_loss jumped to 3.78 on the same architecture and training recipe. That jump wasn't the model getting worse, it was the eval getting honest.

**Lesson 1: Always audit your train/val split. Leakage is silent and optimistic.**

## Phase 2 Problem: NVD Was 87% of the Corpus

With a clean split, Phase 2 trained to 10K steps on what I thought was a diverse cybersecurity corpus. 10,925 records. NVD CVEs, synthetic CTF writeups, synthetic papers.

What I had not measured was token share. NVD CVE descriptions are short (avg ~200 chars) and there are thousands of them. The synthetic writeups are long (~800 chars each) but there were only 500.

Running `scripts/data_stats.py` revealed the actual situation:

```
NVD CVE:     87.3% of tokens
Synthetic:    7.8%
Papers:       4.9%
```

The model was not learning "cybersecurity language." It was learning "CVE description language", one very specific register of one very specific document type.

Every prompt, regardless of domain, produced CVE-style output:

- CTF prompt produced a fake CVE
- MITRE ATT&CK prompt produced a fake CVE
- Research paper prompt produced a fake CVE

The model had one output register and applied it to everything.

**Lesson 2: Measure token share, not record count. A dataset with 6 sources and 87% from one source has 1 effective source.**

## Phase 3: Real Data From Real Sources

Phase 3 added three new real data sources:

- **MITRE ATT&CK**, pulled via STIX2 JSON from GitHub. 691 technique descriptions covering adversary tactics, techniques, and procedures.
- **CAPEC**, an XML pull from MITRE covering attack patterns. 609 records with structured attack methodology descriptions.
- **CTFtime real writeups**, 467 inline writeups from actual CTF events, attributed and licensed for research use.

These three sources brought the token distribution to:

```
NVD CVE:       65.3% (capped at 6M tokens via content-hash subsample)
Synthetic CTF: 17.2%
arXiv cs.CR:    8.4%
CTFtime real:   5.3%
MITRE ATT&CK:   2.9%
CAPEC:          0.9%
```

The NVD cap uses deterministic content-hash subsampling, the same 71,828-record prefix every rebuild, so train/val splits stay byte-identical across runs:

```
python3 scripts/rebuild_corpus.py --max-cve-tokens 6000000
```

Trained ghost-tiny for 30,000 steps on the rebalanced corpus. The per-source perplexity results were striking:

| Source | Before (v0.3.3) | After (v0.3.5) | Change |
|--------|-----------------|----------------|--------|
| MITRE ATT&CK | 615 | 55 | -91% |
| CTFtime writeups | 184 | 61 | -67% |
| CAPEC | 326 | 134 | -59% |
| Synthetic CTF | 68 | 28 | -58% |
| arXiv | 671 | 355 | -47% |
| NVD CVE | 24 | 28 | +14% |

Every diversity source improved dramatically. NVD paid a small cost (+14%), expected, since it now shares parameter capacity with five other domains. That is the right tradeoff.

The behavioral change was visible. Same model size, same training recipe, different corpus balance:

Before (v0.3.3), prompt "MITRE ATT&CK technique T1003" produced "CVE-2019-XXXX: A vulnerability in [product] allows remote attackers to...". After (v0.3.5), the same prompt produced "T1003.011: defense-evasion. Tactic: defense-evasion. Adversaries may use...". That is actual MITRE schema output: sub-technique ID format, "Tactic:" header, the standard "Adversaries may..." opening. The model learned the MITRE register because it finally had MITRE training data in meaningful quantity.

**Lesson 3: Model behavior follows token distribution. Want register diversity? Give your corpus source diversity at the token level.**

## The Evaluation Was Broken Too

While fixing the data, I discovered the evaluation was also broken.

The security task evaluation asked the model to classify inputs into categories (CVE severity, vulnerability type, MITRE tactic) by scoring each candidate label's log-probability. Every model across every phase reported 13.3% accuracy, exactly 4/30, exactly at the random baseline.

That is not the model performing poorly. That is a mode-collapsed eval. The model was assigning its highest probability to the same single label for every sample in each task, the most common token sequence, regardless of the input.

The fix was PMI (Pointwise Mutual Information) scoring:

```python
# Instead of: score = log P(label | context)
# Use: score = log P(label | context) - log P(label)
pmi_score = conditional_logprob - unconditional_logprob
```

This subtracts the model's prior preference for each label, so common labels do not automatically win. After switching to PMI scoring, the eval could finally discriminate between models:

| Phase | PMI Accuracy | vs Random (14.5%) |
|-------|--------------|-------------------|
| Phase 1 | 20% | +5.5 pp |
| Phase 3.5 | 31.2% | +16.7 pp |

Not impressive in absolute terms, ghost-tiny at 14.7M params is not going to nail classification tasks. But it is real signal, not evaluation noise.

**Lesson 4: If every model in every phase scores the same on your eval, the eval is broken. Check for mode collapse before concluding the model isn't learning.**

## Phase 3.6: Finding the Capacity Ceiling

With a working eval and a clean corpus, I tried to push further. Phase 3.6 added Exploit-DB (~3.77M tokens, 30% of the new corpus) and re-trained ghost-tiny at the same 30K-step recipe.

The results were a clean regression:

| Task | Phase 3.5 | Phase 3.6 | Change |
|------|-----------|-----------|--------|
| CVE Severity | 32% | 16% | -16 pp |
| Vulnerability Type | 32% | 12% | -20 pp |
| Attack Technique | 40% | 16% | -24 pp |
| CTF Categorization | 40% | 20% | -20 pp |
| Overall | 31.2% | 16.8% | -14.4 pp |

Per-source perplexity confirmed the diagnosis: every existing source got 28 to 42% worse while Exploit-DB itself landed well (PPL 40.87). The model learned Exploit-DB at the expense of everything else. Parameter capacity was reallocated, not expanded.

At 14.7M parameters and 30K training steps, ghost-tiny has hit its ceiling. More data at this model size produces diminishing returns, and eventually, regression.

**Lesson 5: Parameter capacity is the binding constraint. When adding data hurts existing domains, you have hit the model's capacity wall. The fix is more parameters, not more data.**

## Where GhostLM Is Now

v0.3.5 (current canonical):

- 30,000 training steps
- 74,635 records, 8.8M tokens across 6 sources
- Cyber-text perplexity: 96.24 (vs GPT-2's 26.76, ~8x less capacity, ~3.6x behind)
- PMI security task accuracy: 31.2% (vs 14.5% random)
- Register diversity: switches between CVE, MITRE, and CTF voice depending on prompt

What is next:

- ghost-small (55M params), the first scale-up rung, targeting M4 GPU/MPS
- Corpus volume expansion: CTFtime archives, security research blogs, full-text papers
- Applied for Google TPU Research Cloud credits

The honest framing: ghost-tiny is a learning artifact and a working pipeline. It is not a useful cybersecurity AI tool. ghost-small is where domain-coherent generation might start to emerge. ghost-base (~350M) is where it gets genuinely useful. That is the realistic roadmap.

## What I'd Tell Someone Starting This

1. **Build the data pipeline before the model.** The architecture is a solved problem, transformers are well understood. What is not solved for your domain is the data. Start there.
2. **Measure token share, not record count.** Diversity in records does not mean diversity in training signal if one source dominates token count.
3. **Fix your evaluation before trusting it.** Mode-collapsed evals are optimistic and useless. PMI scoring, per-source perplexity, and fixed external test sets are more honest than aggregate val_loss.
4. **Set a capacity budget and test it.** Before adding more data, test whether your model can absorb it without regressing on existing domains. A simple per-source perplexity check after each training run tells you whether you are within capacity.
5. **Be honest about what your model can't do.** A 14.7M parameter model hallucinating at temperature 0.7 is not a cybersecurity tool. Label it accurately. The trajectory toward useful matters more than the current absolute performance.

## Try It Yourself

```
git clone https://github.com/joemunene-by/GhostLM.git
cd GhostLM
make install
make data
make train-tiny
make chat
```

The codebase is designed to be readable: every component is hand-written with docstrings, and the architecture is clean enough to use as a reference for how a transformer actually fits together.

Contributions welcome, especially around corpus expansion, evaluation design, and architecture improvements (RoPE and Flash Attention are already in, SwiGLU and grouped query attention are next candidates).

GitHub: https://github.com/joemunene-by/GhostLM
License: MIT

Built in Nairobi, Kenya.

---

By **Joe Munene**, a software engineer in Nairobi focused on secure systems and applied machine learning.
[Portfolio](https://my-portfolio-peach-eta-42.vercel.app) · [GitHub](https://github.com/joemunene-by) · [More writing](https://github.com/joemunene-by/writing) · [joemunene984@gmail.com](mailto:joemunene984@gmail.com)
