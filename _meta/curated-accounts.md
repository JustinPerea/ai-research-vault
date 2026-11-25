# Curated accounts

Public X accounts whose posts about a paper count as a **keep signal** for the daily digest. Added 2026-11-25 with Justin's approval; edit this list to add or remove accounts.

| Account | Profile |
|---|---|
| `dair_ai` | https://x.com/dair_ai |
| `_akhaliq` | https://x.com/_akhaliq |

## How they are used

- During a digest run, the pipeline may look up recent public posts from these accounts (about the last two days) and match them to papers in that day's candidate pool (arXiv cs.AI, cs.CL, cs.LG and the Hugging Face Daily Papers top 10).
- A mention is a reason to **keep a borderline paper**, one that is close to the stack filter but not clearly in. The item notes the mention (account and date).
- A mention is **never** a reason to drop a paper, and never admits a paper that fails the stack filter outright or that is not in the candidate pool.
- A mention adds no facts: the item is still written from the arXiv abstract only.
- The signal is optional. If X lookups fail or are not worth the cost that night, the run skips it and says so in Sources checked.
- Rescues and lookup failures are logged in the digest's Sources checked.

See [pipeline.md](pipeline.md) for the full rules.
