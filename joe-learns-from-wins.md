# I built a local coding agent that learns from its wins, not just its mistakes

Most agents handle memory in one of two ways. Either they forget everything between sessions, or they "learn" by fine-tuning on a pile of past conversations and hoping the gradient sorts it out. I wanted something narrower and more honest for joe, the local-first agent shell I have been building: learn from the sessions that actually worked, and turn each one into a reusable skill I can read, edit, or delete.

This post is about the feature I just shipped to do that, and why I think the design matters more than the feature itself.

## What joe is, quickly

joe is a terminal coding agent, in the spirit of Claude Code, except every model runs on my own GPU through ollama and every byte of state lives in `~/.joe-agent/` on my machine. It has the usual tools (read, write, edit, shell, grep, web), a planner, and a separate coder model it delegates to. Nothing leaves the laptop.

The part I care about is that joe is supposed to get better the longer I use it, without me retraining anything. It already learned from corrections: every time I hit `/undo`, that is a signal, and a background loop distills recent corrections into short preference rules that get injected into future prompts. Correction in, behavior change out.

The gap was the other half. joe learned from everything I rejected, and nothing from what I accepted.

## Learning from wins

The new feature is skill synthesis. After a multi-step session that actually worked, I run one command and joe reads the full transcript of that session and decides whether it contains a generalizable procedure worth keeping. If it does, it writes a skill: a small Markdown file with a name, a description, a set of trigger keywords, and the reusable steps written as instructions to a future agent. If the session was just chatter or a one-off edit, it returns nothing. Not every session deserves to become a skill, and the model is told to say so.

The skill lands in `~/.joe-agent/skills/`, and from then on, whenever a future request matches its triggers, joe injects it into the prompt automatically. So a workflow I figured out once (the right sequence of steps to do a tricky migration, say) is available the next time I ask for something similar, without me remembering to mention it.

The important detail: skills are plain text. I can open one, fix a wrong step, or throw it away. There is no opaque weight update to debug. If a skill is bad, I delete a file.

## Why this design and not fine-tuning

The idea is not mine. It comes from Voyager, the Minecraft agent that built an ever-growing library of executable skills and used it to compound its abilities without touching the model weights. The Voyager result that stuck with me is that a skill library gives you genuine lifelong learning and sidesteps catastrophic forgetting, because adding a new skill never degrades the old ones. A new text file cannot make the model worse at something else. A fine-tune can.

For a local setup that matters even more. I am running small models on consumer hardware. I cannot afford to retrain every time I learn something, and I cannot afford the regressions that come with it. A skill library is cheap, interpretable, and reversible. It fits the constraints honestly instead of pretending I have a datacenter.

## The honest part

This is new and it is not magic. The quality of a synthesized skill depends on the orchestrator model that writes it, and on a small local model the output sometimes needs a human edit before it earns its place. I made synthesis manual on purpose, one command rather than an automatic background step, because I do not want skills appearing without me seeing them, and joe's whole stance is that skills are suggestions injected into context, never code that runs on its own.

The next problem is the interesting one. Right now joe can write a skill, but it cannot yet tell whether that skill actually helped. The signal is already in the system: a turn where a skill was injected and I did not hit `/undo` is a quiet win, and one I corrected is a quiet loss. The next thing I am building is the ledger that tracks this, so joe can show me which skills earn their place and retire the ones that do not. That closes the loop: write from wins, measure against corrections, prune what does not work.

That combination, a skill library that knows its own track record, running entirely on local hardware, is the part I have not seen elsewhere. It is the reason I am still building this instead of just using a hosted agent.

joe is open source: https://github.com/joemunene-by/joe
