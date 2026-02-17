Before we get into my contributions, here is a [link](https://www.researchgate.net/publication/358172152_Design-by-Analogy_Effects_of_Exploration-Based_Approach_on_Analogical_Retrievals_and_Design_Outcomes) to the paper that my mentor/adviser made on the original iteration of this project. 

<h3>Intro / Goals</h3>

  

The VISION+ project uses a large language model to pull out important concepts from patent text and builds interactive network visualizations so you can see how patents relate through 4 key lenses: components, behaviors, engineering principles, and functions. The version I worked on (VISION+AI) takes the existing pipeline and ports it to run on Ollama with Gemma locally. The goal was to keep the same analytical “lenses” and visualizations while moving the brain of the system to a local only model. It also adds the option to point the script at a remote Ollama instance (e.g., over Tailscale) when I want to run analysis from a laptop against a stronger machine. This write-up summarizes what the tool does and the changes I made to get there.


<h3>What VISION Does: Four Lenses and a Graph</h3>


The system treats each patent as a bag of concepts and then connects patents to each other through the four analytical “lenses”:

<table>

<thead>

<tr>

<th>Lens</th>

<th>Focus</th>

<th>Example terms</th>

</tr>

</thead>

<tbody>

<tr><td><strong>Component</strong></td><td>Physical parts, tangible objects</td><td>gear, cable, sensor, motor, housing, spring</td></tr>

<tr><td><strong>Behavior</strong></td><td>Properties and characteristics</td><td>elastic, conductive, thermal, rigid, flexible</td></tr>

<tr><td><strong>Engineering principle</strong></td><td>Core technical concepts</td><td>efficiency, stability, resonance, friction, leverage</td></tr>

<tr><td><strong>Function</strong></td><td>Actions and operations</td><td>rotate, connect, transmit, filter, control</td></tr>

</tbody>

</table>

A local LLM (Gemma, via Ollama) reads patent text and, for each lens, returns a short list of terms plus importance weights (0.0–1.0). Those are stored in SQLite; a separate step builds force-directed graphs (D3.js) that link keywords to patents. So you get four interactive visualizations—one per lens—with filtering by keyword and CPC-style group, search, zoom, and double-click to highlight neighbors. 

<h2>Changes Overview</h2>
I took on VISION when the analysis pipeline was still driven by non-negative matrix factorization (NMF)—i.e. no LLM or other neural/AI component. All of the AI-based analysis in this project is work I added: the move from NMF to LLM-powered keyword extraction, the Claude-based version (VISION+CLAUDE), and the local Ollama/Gemma version (VISION+AI), including the prompts, response parsing, and integration with the existing database and visualizations. The four-lens framework and the visualization stack were already part of VISION; the shift to using an LLM to populate those lenses and the design of that pipeline are mine.
<h3>From Claude to Gemma</h3>

**Replacing the LLM backend**

- **New class: `OllamaPatentAnalyzer`**

- Talks to Ollama’s `/api/generate` (prompt + model name, `stream: false` for a single response).

- On startup, hits `/api/tags` to check that the chosen model (e.g. `gemma3:12b`) is present and prints clear errors if the server is down or the model is missing.

- Uses a 6,000-character window of patent text per request (smaller than the 8,000 used for Claude) to stay within local model context limits.

- Prompts tuned for Gemma

Gemma tends to do better with very explicit, task-style instructions. I rewrote the per-lens prompts into a strict “Task / Instructions / Example output” format and added a hard constraint that terms must be one word to keep parsing reliable. Each prompt spells out the expected line format: `term score` with score in [0.0, 1.0].

  

- Robust model output parsing

Local models don’t always stick to the format. I added `parse_llm_response` to more proactively skip lines that look like task text, separators, or bullet/numbered lists. Duplicates are removed and the list is capped at 15 terms per lens so the rest of the pipeline stays consistent.

  

**Remote Ollama and Tailscale**

  
- I didn’t want to be tied to running the script only on the machine that runs Ollama. So the script accepts `--ollama-url` and a convenience flag `--tailscale-ip`. If you pass `--tailscale-ip 100.x.x.x`, it sets the Ollama base URL to `http://100.x.x.x:11434`. That way I can run the analysis on a laptop (or other thin client) while Ollama and the heavy model sit on a desktop or server on the same Tailscale network. The README and in-code comments mention this as an option.

  

**CLI and docs**

  

- The argument parser already supported `--model`, `--batch-size`, `--visualize`, `--lens`, `--top-keywords`, `--top-patents`, etc. I added `--ollama-url` and `--tailscale-ip`, and wired them through so the analyzer uses the chosen base URL.

- I wrote the VISION+AI README (in `VISION+AI/README`): environment setup (venv, pip), Ollama install and `ollama pull gemma2:9b` (or gemma3:12b), patent file layout (`patents/*_all.txt`), the three-phase workflow (import → analyze → visualize), full CLI reference, design notes (local-only data, modular components, incremental processing), usage examples, and troubleshooting (connection, memory, speed, visualization size). The root `README.md` was updated to point to the current version (VISION+AI) and to the older variants (MatLab, LDA, CLAUDE) in OldVersions.

  

<h3>Tech Snapshot</h3>

  

<table>

<thead>

<tr>

<th>Layer</th>

<th>Choice</th>

</tr>

</thead>

<tbody>

<tr><td>Language / runtime</td><td>Python 3 (venv), argparse for CLI</td></tr>

<tr><td>Database</td><td>SQLite (patents, patent_keywords, visualizations)</td></tr>

<tr><td>LLM</td><td>Ollama + Gemma (default <code>gemma3:12b</code>), configurable URL</td></tr>

<tr><td>Visualization</td><td>D3.js (force-directed graph), jQuery UI (search/filters), HTML/JSON output</td></tr>

<tr><td>Data flow</td><td><code>patents/*_all.txt</code> → import → analyze (Ollama) → visualize → <code>output/*.html</code> + <code>*.json</code></td></tr>

</tbody>

</table>  

<h3>Design Choices</h3>


- **Local-first**

All patent text and extracted keywords stay on your machine (or your Tailscale network). No cloud API and no API keys for the main analysis path. The trade-off is runtime, but it also provides developers and users with greater individual control.

  

- **Modular layout**

Database, importer, analyzer, and visualization are separate classes. Swapping Claude for Ollama only touched the analyzer and the CLI that constructs it; the rest of the pipeline stayed the same.

  

- **Incremental runs**

The DB tracks which patents are already analyzed. You can add new `*_all.txt` files, re-run import and analyze, and only new patents get sent to the LLM; then you regenerate visualizations as needed.

  

<h3>Workflow / Outcome</h3>

  

In practice I run: (1) `--import` once per collection, (2) `--analyze` with a batch size that fits my RAM (and optionally `--tailscale-ip` when I’m not on the Ollama host), then (3) `--visualize` to refresh the four lens graphs. The interactive HTML lets me filter by keyword and CPC-style group, search for a patent or term, and double-click a node to see its neighborhood—and shift-click still opens the patent on Google Patents. So the outcome is a local, privacy-preserving version of the same high-level workflow, with the flexibility to run the script anywhere and point it at a remote Ollama instance when that’s useful.

  

<h3>Miscellaneous / Addendum</h3>

  

- **Repo layout**

The current version lives in **VISION+AI** (Ollama/Gemma). **OldVersions** holds the original MatLab app, VISION+ (LDA), and VISION+CLAUDE (Anthropic). The root README summarizes this.

  

- **Attribution**

The four-lens framework, database schema, and D3-based visualization design come from the existing VISION project. My contributions are the Ollama/Gemma backend, the Tailscale-friendly remote URL option, the stricter prompts and parsing for local models, and the VISION+AI README and root README update.

  

<h2>Conclusion</h2>
This project is another that I am proud of when it comes to research, but I feel that I unfortunately had to leave this project in a slightly unfinished state. The ultimate goal was to integrate an LLM chatbot to allow users to describe their ideas in plain text and allow the LLM to determine what keywords or patents could provide inspiration or otherwise discourage the undue efforts in pursuing an idea that may already have patents on file. 
**UPDATE AS OF FEB 17TH:** I knew that this project was planned to continue development sans-myself given my graduation and I just recently found a research board that seems to indicate my colleague Antony was able to get this project to completion! I've linked it below for your own academic advancement enjoyment. :)


![[antonyPoster.pdf]]