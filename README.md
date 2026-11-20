# Kuku Promo Finder (Problem 1)

Reads a folder of microdrama screenplays and finds 20 to 60 second promo clips that open on a hook, end on a cliffhanger and make sense to a stranger. It runs fully on a laptop with a free local model (Ollama). Nothing is sent to a paid API.

Everything is in one notebook: **`promo_finder.ipynb`**.

## How it works

| Step | Who | What |
|---|---|---|
| 1. Parse | code (regex) | every screenplay → numbered lines with start/end seconds |
| 2. Shrink | code | drop CAMERA lines (half the script), keep dialogue, action and sound. Each episode is under ~2k tokens |
| 3. Find | AI, one call per episode | picks up to 3 promos as **line numbers**: hook line → cliffhanger line, plus a reason and scores |
| 4. Check | code | line numbers → timestamps. Rules: 20 to 60s, real lines, no overlaps. If a rule breaks, the model is told exactly what's wrong and gets one retry. What's still broken keeps a flag |
| 5. Rank | code | hook + cliffhanger + standalone scores, −5 per flag, +2 for episodes 1 to 10. Overlapping picks are removed |
| 6. Cold-viewer test | AI | the top 20 clean promos are shown to a fresh call with **no context at all**, like a stranger on TikTok. It says what happens, whether it makes sense and what question the ending leaves. Confusing clips drop. The top 10 has at most 2 per episode |
| 7. Hook pass | AI + code | for the top 10, the model re-picks the opening line from the clip plus the 6 lines before it. Code keeps it only if the clip stays 20 to 60s |
| 8. Human | editor | `show_clip(id)` prints the exact script lines of any promo, so a person can check before posting |

**Why line numbers instead of timestamps?** The first version asked the model for timestamps and quotes directly. On episodes 1 to 10 only **1 of 18** promos passed the rules: too short, the hook not at the start, the cliffhanger not at the end, made-up quotes. See `outputs/experiments/v1_model_gives_timestamps/` in the shared outputs folder. Small models judge drama fine but can't do the time maths, so code does the maths now.

## Run it on your laptop (Ollama)

```bash
# 1. model (free, about 4.7 GB)
brew install ollama            # or download from ollama.com
OLLAMA_CONTEXT_LENGTH=8192 ollama serve      # leave this running in its own terminal
ollama pull qwen2.5:7b

# 2. python
python3 -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt notebook

# 3. run
jupyter notebook promo_finder.ipynb          # then Run All
```

Put the screenplay folder next to this repo, or change `FOLDER` in the first code cell. Results go to `outputs/<show>/<model>/`.

The full 50-episode run takes about an hour on an M2 MacBook. Most of that is the `PAUSE = 15` rest after each call, which keeps the laptop cool; set it to `0` if you don't mind the heat. Re-running is instant because replies are cached in `raw/`. Set `USE_CACHE = False` for a fresh run.

## Run it on Google Colab

Colab can't reach the Ollama on your laptop, so install Ollama inside Colab (a T4 GPU runtime is free):

```python
!curl -fsSL https://ollama.com/install.sh | sh
!nohup env OLLAMA_CONTEXT_LENGTH=8192 ollama serve > ollama.log 2>&1 &
!sleep 5 && ollama pull qwen2.5:7b
!pip install -q openai pydantic pandas tabulate
```

Upload the screenplay folder, set `FOLDER = "/content/<folder name>"`, set `PAUSE = 0` and Run All.

## A new show

Change `FOLDER` to the new show's folder and Run All. Output goes to a new folder, so nothing is overwritten. The parser expects the same screenplay format as the input pack: a `**NAME (emotion):**` line, then `Start: (MM:SS.ss) | End: (MM:SS.ss)`, then the text.

## A different model

The notebook talks to the model through the `openai` library, so any OpenAI-compatible server works. Change three lines:

```python
BASE_URL = "http://localhost:11434/v1"
API_KEY = "ollama"
MODEL = "qwen2.5:7b"      # e.g. "llama3.2:3b" for a smaller local model
```

`JUDGE_MODEL` can be set separately, so a different model can do the cold-viewer test. Each model writes to its own output folder, and the last cell compares runs.

## Outputs (per show and model)

- `ranked_promos.md`: the 10 to post first, the cold-viewer table and the full ranking, all readable
- `ranked_promos.csv`, `all_promos.json`, `cold_viewer_test.csv`: the same data for other tools
- `run_info.json`: counts of flags, retries and confusing clips
- `raw/`: every model reply, unedited

## Limits (honest)

- It reads the **script, not the video**. Real pacing, acting and visuals may make a clip better or worse.
- Promos come from **one episode at a time**. Joining moments across episodes isn't done yet.
- The local 7B model's taste is average. It sometimes picks a flat hook, or ends on a quiet line. The rules and the cold-viewer test catch some of this, not all. A stronger model is a 3-line swap.
- The input pack is confidential, so screenplays and outputs are **not** in this repo (`.gitignore`).
