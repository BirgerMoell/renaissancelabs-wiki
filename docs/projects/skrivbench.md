# Skrivbench: a benchmark for natural Swedish writing

**Status:** early, built; calibration against human judges in progress
**Authors:** Renaissance Labs

Publishers and authors tell us that language models write Swedish that is correct but not natural. Typical problems are anglicisms ("svengelska"), stilted sentence rhythm ("stolpigt"), and a generic machine tone ("maskinprosa"). Existing benchmarks do not measure this, so we are building one.

## Design

- **A rubric** with six language dimensions plus overall quality and task fit.
- **48 writing prompts** across genres.
- **A pairwise judge:** a language model reads two answers to the same prompt, in both orders, and picks the better one. Results are aggregated with a Bradley–Terry model.
- **An absolute rubric score,** where the judge must quote the exact spans that cost points.
- **A minimal-pair probe:** 43 pairs of sentences that differ only in naturalness.
- **Lexicon detectors** that flag known anglicisms and calques cheaply.
- **A local annotation tool** for human labels, and a script that checks how well the judge agrees with them.

## Where it stands

- The rubric, prompts and minimal pairs were drafted with Claude and are **not yet verified by native-speaker language professionals**. They are marked unverified in the data.
- A smoke test with two small models and an Opus judge ran end to end.
- The minimal-pair probe is close to ceiling for the Claude models we tried (93–98%), so it separates weak models from strong ones but is not a good ranking tool at the top.
- The next step is human labelling: minimal pairs first, then about a hundred blind pairs, to calibrate the judge before we trust any ranking.

## Why human labels matter

An LLM judge can prefer its own style, or prefer the kind of Swedish that models write. The point of calibrating against people who write for a living is to check that the benchmark measures what readers hear and not what the judge likes.

## Wanted

If you are an author, translator, editor or linguist and want to label pairs or add hard cases, see [Contributing](../contributing.md).
