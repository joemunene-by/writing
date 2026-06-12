# Writing

Technical writing on AI security, machine learning systems, and engineering, by **Joe Munene**, a software engineer in Nairobi focused on secure systems and applied ML.

[Portfolio](https://my-portfolio-peach-eta-42.vercel.app) · [GitHub](https://github.com/joemunene-by) · [dev.to](https://dev.to/ghost_gi_m) · [joemunene984@gmail.com](mailto:joemunene984@gmail.com)

I write from things I actually build: [GhostLM](https://github.com/joemunene-by/GhostLM), an 81M-parameter cybersecurity language model trained from scratch, and [joe](https://github.com/joemunene-by/joe), a local-first agent shell. The pieces below are the lessons that were worth writing down.

## Articles

| Piece | Topic |
|-------|-------|
| [I built a local coding agent that learns from its wins](joe-learns-from-wins.md) | Skill synthesis in [joe](https://github.com/joemunene-by/joe): turning successful sessions into reusable, editable skills (the Voyager skill-library idea, local-first), and why that beats fine-tuning for a small on-device model. |
| [Evaluating a small LLM on multiple choice without fooling yourself](evaluating-small-llms-without-fooling-yourself.md) | Seven ways an MCQ benchmark lies to you at small scale (letter-scoring bias, permutation debiasing, PMI, split leakage, contamination, per-source perplexity, free-form recall) and the fix for each. Drawn from building and debugging GhostLM. |
| [AI Model Supply Chain Security](ai-model-supply-chain-security.md) | Serialization risks in model artifacts, provenance verification, artifact scanning, and Model Bills of Materials. Originally proposed to the OWASP Cheat Sheet Series ([PR #2111](https://github.com/OWASP/CheatSheetSeries/pull/2111)). |

## License

Articles are licensed [CC BY 4.0](LICENSE); code snippets within them are also available under MIT. Share freely with attribution.
