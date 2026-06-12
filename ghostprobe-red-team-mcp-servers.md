# I built a red-team scanner for MCP servers. Then I pointed it at the real ones.

The Model Context Protocol lets an AI agent connect to external tools: a filesystem, GitHub, Slack, a database. Each server an agent connects to advertises a list of tools, and here is the part most people miss: that tool list is an attack surface.

A tool's description and parameter docs do not just describe the tool to a human. They are injected straight into the agent's context, and the model treats them with the same authority as your own instructions. So a server can hide instructions to the model inside text you skim as a harmless description. That is tool poisoning, and it is the pattern behind CVE-2025-54136.

I wanted to see how exposed real servers were, so I built ghostprobe: a scanner that connects to an MCP server, pulls its tool list, and reports what an attacker would care about, mapped to the OWASP MCP Top 10. This post is about what happened when I pointed it at real servers, because that is where it got interesting.

## What the tool list gives away

ghostprobe looks for a few things:

- Tool poisoning: instruction-injection phrasing like "ignore previous instructions" or "do not tell the user", and invisible Unicode used to smuggle instructions past human review while still reaching the model.
- The lethal trifecta: a server whose tools together provide private-data access, a way to send data out, and exposure to untrusted content. Any one leg is fine. All three, and a single prompt injection can read your secrets and exfiltrate them. The term is Simon Willison's.
- Rug pulls: a server that silently changes a tool's description after you have trusted it.
- Dangerous capabilities like shell execution.

The lethal trifecta is the one that matters most, because it is not about a single malicious tool. It is about a combination that looks reasonable until you see all three legs at once.

## I pointed it at the official servers, and it found bugs in itself

The official reference servers (filesystem, memory, GitHub, and the rest) are well-built, so I expected clean results. Instead, the first things ghostprobe found were false positives in my own analyzer. This was the most useful thing that could have happened.

On the filesystem server it flagged `read_text_file` as code execution. Wrong. It matched because my exec detector keyed on the bare word "system", and the description says "file system". A read-a-file tool is not remote code execution.

On the sequential-thinking server it flagged a thought history as private-data access. Wrong again. It matched the word "history".

A security scanner lives or dies on its false-positive rate. A tool that cries wolf gets muted, and a muted tool is worse than no tool. So I fixed both: exec detection now requires a real execution verb plus an object, and "history" is too weak a signal to keep. Each fix shipped with a regression test built from the exact server that exposed it.

## Then GitHub exposed a false negative, which was worse

The GitHub server was more interesting. ghostprobe saw that it reads private repo contents and reads issue text, but it reported no sink, so no trifecta. That was a false negative, and a false negative in a security tool is more dangerous than a false positive.

The GitHub server absolutely has a sink. It can create issues, post pull-request comments, and push to a repo. Writing into a public issue is how you exfiltrate private data. ghostprobe missed it because GitHub's write verbs are "create" and "post" and "push", not "send", and I had only taught it to recognize sending over an obvious external medium like email or HTTP.

The fix was to recognize that writing to a shared, remote, collaborative service is itself an exfiltration channel, distinct from writing a local file. Open an issue, post a comment, push to a repo: those send data out of your trust boundary. Writing a local file does not. After that change, ghostprobe flagged the GitHub server correctly:

```
[CRIT] MCP04 Lethal Trifecta
  data:      get_file_contents, get_pull_request_files, push_files
  sink:      add_issue_comment, create_issue, create_or_update_file
  untrusted: get_issue, get_pull_request_comments, list_issues
```

Read a private repo, ingest issue text that anyone on the internet can write, and post to a public issue. An attacker files an issue containing instructions, an agent set up to triage issues reads them, and your private code ends up in a public comment.

I want to be precise about what this is. It is not a vulnerability I discovered. This exact GitHub-MCP exfiltration path was disclosed by Invariant Labs in 2025. The point is that ghostprobe detects the class automatically, from the tool list alone, with no prior knowledge of the server. Point it at a server it has never seen and it tells you whether the trifecta is present.

## What I actually learned

Real servers are the only real test. Every meaningful improvement to ghostprobe came from running it against an actual server, not from my fixtures. A handful of releases in two days, each one a precision fix driven by a real result: two false positives removed, one false negative closed.

The tool list is underappreciated as an attack surface. People audit MCP server code. Far fewer look at what the server advertises to the agent, which is exactly where poisoning and the trifecta live, and which an attacker can influence without touching your code at all.

And a scanner's credibility is its false-positive rate, not its feature count. The most valuable work was not adding checks. It was making the existing checks stop lying.

## Try it

ghostprobe is open source under MIT. The analyzer has no dependencies; you only need the MCP SDK to probe a live server.

```
pip install "git+https://github.com/joemunene-by/ghostprobe.git" mcp
ghostprobe stdio -- npx -y @modelcontextprotocol/server-github
```

Or scan a saved tools/list dump completely offline:

```
ghostprobe scan-file tools.json
```

GitHub: https://github.com/joemunene-by/ghostprobe

---

By **Joe Munene**, a software engineer in Nairobi focused on secure systems and applied machine learning.
[Portfolio](https://my-portfolio-peach-eta-42.vercel.app) · [GitHub](https://github.com/joemunene-by) · [More writing](https://github.com/joemunene-by/writing) · [joemunene984@gmail.com](mailto:joemunene984@gmail.com)

Discussion: [Coder Legion](https://coderlegion.com/20451/i-built-a-red-team-scanner-for-mcp-servers-then-i-pointed-it-at-the-real-ones)
