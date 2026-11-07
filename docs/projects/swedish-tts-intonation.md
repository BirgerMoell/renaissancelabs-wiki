# Fixing Swedish intonation by editing weights

**Status:** finished (first round)
**Authors:** Renaissance Labs

The open text-to-speech model Supertonic 3 speaks Swedish, but its intonation sounded odd, and flatter than its English. We tried to fix that by editing a few weights directly, without training. We were inspired by the blog post [Weight-Space Programming](https://rikiyat.github.io/blog/weight-space-programming).

## What we found

**It was not pitch range.** Swedish and English had the same overall pitch spread. The problem was local: how phrases start and end, and where pauses fall.

**The model resets pitch at full stops but not at commas.** After a period it pauses and the pitch jumps back up, so each sentence sounds like its own phrase. After a comma it only flattens slightly. Text without punctuation becomes one long downward slope, which is the "weird" shape we kept hearing.

**The model has no language switch.** Swedish and English share every weight. The language is only a tag on the text, so any edit changes both.

## The edit that worked

We blended the embedding row for `,` toward the row for `.`, with a weight of 0.7, in two places: the text encoder and the duration predictor. That is a small change to two embedding tables and nothing else.

The result: commas became real phrase boundaries, with pitch resets and short pauses. Word error rate, measured with a speech recogniser, did not change. Listening tests agreed: a listener called the edited model "clearly much better" than the original.

A small front-end step helps further. It inserts commas before conjunctions such as *men*, *medan* and *eftersom*, so unpunctuated text also gets phrase resets.

## What did not work

- **Scaling the attention outputs** changed pitch movement a little and did nothing useful.
- **Steering the voice style** toward livelier voices made the speech breathy.
- **Slowing the speech** with the speed setting made it sound worse, even though it moved pauses closer to a commercial reference voice.
- **Automatic quality scores** did not match listening. The MOS predictor we tried (UTMOS22) preferred our model over the commercial reference in 29 of 30 pairs, which is not credible, so we used it only as a guard against artefacts.

## A lesson about metrics

The same pattern showed up again when we looked at another open Swedish voice that had a faint echo in its output. Filters that removed the echo in a cepstral measurement did not remove it for a listener. A weight edit, moving the decoder 75% toward the base English model, did. We now treat a listening test as the final check and use metrics only to find candidates.

## Related

We applied the same approach to a Swedish speech recogniser (a 600M-parameter model). Raising the output bias for two tokens fixed a systematic problem where the model never wrote "procent" in dictation. Test error on real speech was unchanged. Edits that tried to improve things in general did not hold up on held-out data.
