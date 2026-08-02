# ACE-Step Song Generator

Suno-style song generator — style tags + lyrics in, full song out — running
ACE-Step (open source, vocals, 19 languages incl. Spanish and French) on a
hosted GPU. Nothing to install.

## The permanent address

**https://ace-step-song-generator.vercel.app** — bookmark this. It never
changes.

That page is `index.html` in this repo, deployed on Vercel (project
`ace-step-song-generator`, auto-deploys on every push to `main`). On load it
fetches `link.txt` from this repo and points its button at whatever URL is in
there. `index.html` also carries a hardcoded fallback used only if that fetch
fails. Both currently hold:

```
https://ace-step-ace-step.hf.space/
```

That is the public HuggingFace Space `ACE-Step/ACE-Step`, badge "Running on
ZERO". It works anonymously, with no login, and it is always on. To repoint the
page at a different backend, edit `link.txt` — alone on the first line, nothing
else to do, no redeploy. Raw GitHub caches for 300s, so it takes up to about
five minutes to take effect.

## Free Colab does not work. Do not retry it.

The original version of this repo ran ACE-Step on Colab's free T4 and published
a `gradio.live` tunnel. That path is closed, for three measured reasons:

- **`--bf16 true`** loads and runs diffusion, then dies at the final decode with
  `RuntimeError: GET was unable to find an engine to execute this computation`.
  cuDNN has no bf16 conv engine for ACE-Step's DCAE decoder on a T4.
- **`--bf16 false`** (fp32) is OOM-killed by the memory cgroup while loading
  weights: `Memory cgroup out of memory: Killed process 21631`. RAM peaks around
  11.3 GB of 12.9 GB with the GPU still at 3 MiB, so `--cpu_offload false`
  doesn't rescue it either.
- **There is no fp16 flag.** The full list is `--checkpoint_path --server_name
  --port --device_id --share --bf16 --torch_compile --cpu_offload
  --overlapped_decode`.

Separately, Colab's FAQ prohibits "file hosting, media serving, or other web
service offerings not related to interactive compute" and "bypassing the
notebook UI to interact primarily via a web UI", so a persistent tunnel was
never a legitimate long-term answer regardless.

## Driving it by API

The Generate button is unreliable through tunnels; the HTTP API is not.

```javascript
// 1. discover the signature
const j = await (await fetch('/gradio_api/info')).json();
const P = j.named_endpoints['/__call__'].parameters;

// 2. submit
const data = P.map(p => p.parameter_default);
data[0] = 30;   // audio_duration
data[1] = '<style tags>';
data[2] = '<lyrics>';
data[3] = 27;   // infer_step
data[4] = 15;   // guidance_scale
const r = await fetch('/gradio_api/call/__call__', {
  method: 'POST',
  headers: {'Content-Type': 'application/json'},
  body: JSON.stringify({data})
});
const eventId = JSON.parse(await r.text()).event_id;

// 3. read the SSE result stream
const res = await fetch('/gradio_api/call/__call__/' + eventId);
// event: complete -> data[0].url is the finished mp3
```

The Space's `/__call__` takes **22** parameters and has **no** `format`
parameter. The old Colab build took 24, with `format` at index 0, so every index
shifts by one between the two. Don't reuse indices across builds.

Output files live in the Space's `/tmp` and are ephemeral. Download anything
worth keeping.

## Embedding this in another app

Don't point end users at the shared Space. ACE-Step is sold as a hosted API —
fal.ai (`fal-ai/ace-step`) and Replicate (`lucataco/ace-step`) both run it for
roughly three cents a song. The right shape is a serverless function that holds
the API key server-side and proxies the call.
