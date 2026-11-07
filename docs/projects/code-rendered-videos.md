# Making explainer films from code

**Status:** ongoing
**Authors:** Renaissance Labs

We make long-form explainer films without a video editor. A film is a script file, a text-to-speech step, and a set of React components that draw each scene. One command turns them into an MP4. Changing a sentence means editing the script and re-rendering only the scenes it touches.

## The pipeline

1. **`script.json`** holds the narration, one entry per scene.
2. **A build script** reads the narration, generates speech locally with the open-source Kokoro 82M model, and writes one audio file per line. It also writes a `timings.json` with the exact duration of each line, and a generated ambient pad underneath.
3. **Scenes** are [Remotion](https://www.remotion.dev/) components. Every animation reads its timing from `timings.json`, so visuals stay locked to the voice.
4. **A render step** produces a 1080p H.264 file. Long films take from 20 minutes to about an hour on a laptop.
5. **Review stills:** a script renders one still per narration line and stitches them into a contact sheet, so we can check a whole scene at a glance instead of watching it.

## Things we learned

- **Scene components need a wrapper.** The hook that gives a component its scene timing only works inside a context provider, so each scene is wrapped (we call it `withScene`) and the hook is called inside, never in the outer component.
- **Keep content out of the caption area.** Captions take the bottom ~170 px. Anything important stays above about y = 900.
- **Pronunciation needs a map.** Local TTS mispronounces names and acronyms, so the build script has a respelling table for them.
- **Render at modest concurrency, with a guard.** A long render can use all the memory on a laptop. We render one film at a time at low concurrency, and a small watchdog kills the render if free memory drops below 12%.
- **Label schematic charts as schematic.** If a chart illustrates an idea rather than plots data, the film says so on screen.
- **Check every number against sources before it goes into narration.** Each film has a `research/DOSSIER.md` listing its claims, sources and our confidence in each.

## Films made this way

- A film on what the strongest open-weights AI models have in common (13 min)
- A film on how frontier models are trained: data, scaling, chips, failures, post-training and cost
- A film on hardware limits for AI accelerators
- A psychology series on time: past, present and future, and how experience is shaped by interpretation
- Practical films on attachment, burnout and emotional regulation

## Writing for the audience

For the psychology films we found that a literature review is the wrong shape. The first attachment film had many papers and caveats and was too academic for its viewers. The second version leads with recognition ("this is what it feels like"), gives concrete phrases and steps, and keeps the research to a one-line note on screen plus a sources slide.

## Reproducing it

The pieces are all open: Kokoro for speech, Remotion for rendering, `ffmpeg` for audio. The project-specific parts are small: the script format, the timing file, and the scene wrapper described above.
