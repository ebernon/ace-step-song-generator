# ACE-Step Song Generator (free, on Google Colab)

Suno-style song generator — style tags + lyrics in, full song out — running
ACE-Step (open source, vocals, 19 languages incl. Spanish and French) on
Google Colab's free T4 GPU. Nothing to install locally.

## The permanent address

**https://ace-step-song-generator.vercel.app** — bookmark this, not the
`gradio.live` link. It never changes.

That page is `index.html` in this repo, deployed on Vercel (project
`ace-step-song-generator`, auto-deploys on every push to `main`). On load it
fetches `link.txt` from this repo and points its button at whatever URL is in
there.

**To point it at a new session:** edit `link.txt` in this repo and put the new
`https://xxxxx.gradio.live` URL in it, alone on the first line. Nothing else to
do; no redeploy needed. The page picks it up within about 5 minutes (raw
GitHub caches for 300s). `index.html` also carries a hardcoded fallback URL
used only if the fetch fails.

## How to run

1. Open `ACE_Step_Song_Generator.ipynb` in Google Colab
   (easiest: https://colab.research.google.com → GitHub tab → paste this repo URL;
   a copy also lives in Eric's Google Drive as "ACE-Step Song Generator").
2. Runtime > Change runtime type > **T4 GPU**.
3. Run the cells top to bottom.
4. The last cell prints a public link like `https://xxxxx.gradio.live` —
   open it on any phone or browser. That's the generator.
5. Paste that URL into `link.txt` in this repo so the permanent address above
   points at the new session.

## The cells (for reference / manual rebuild)

```
# Cell 1 — confirm GPU
!nvidia-smi

# Cell 2 — install ACE-Step
!git clone https://github.com/ace-step/ACE-Step.git && cd ACE-Step && pip install -e . -q

# Cell 3 — REQUIRED FIX: Colab ships gradio 6.x, which breaks ACE-Step's UI
!pip install -q gradio~=5.0

# Cell 4 — launch with a public share link (bf16 OFF: T4 cuDNN cannot run the bf16 decoder)
!acestep --share true --bf16 false --torch_compile false --cpu_offload true --overlapped_decode true
```

## Notes / gotchas (learned the hard way, 2026-07-22)

- **gradio must be 5.x.** Without Cell 3 the launch dies with
  `Audio.__init__() got an unexpected keyword argument 'show_download_button'`.
- The ~7GB model weights load at the **first generation**, so the first song
  takes several minutes; subsequent songs are faster.
- Free Colab sessions last a few hours. **Download songs before closing** —
  the runtime's files are wiped. Restart = rerun the cells (fresh gradio.live URL).
- **bf16 must be false on the T4.** With `--bf16 true` the UI loads and diffusion runs, but every generation dies at the final decode with `RuntimeError: GET was unable to find an engine to execute this computation` (cuDNN has no bf16 conv engine for the DCAE decoder on T4). fp32 is slower but works.
- Colab's cell editor mangles multiline pasted/typed shell commands
  (autocomplete eats URLs); keep cells as single-line `&&`-joined commands.
