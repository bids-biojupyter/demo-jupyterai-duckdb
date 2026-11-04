# demo-jupyterai-duckdb
DuckDB experiments on JupyterHub, Jupyter AI, and OpenCode skill

Natural-language querying of a DuckDB database from the Jupyter AI chat panel on JupyterHub,
using an agent (OpenCode) steered by a **skill** which is a Markdown file that tells the
agent what the data means and how to query it well.

## How it works 

- User asks questions in Jupyter AI ──► ACP Persona ──► OpenCode agent ──► reads SKILL.md: tables, query templates ──► DuckDB ──► Answer in chat ──► generate a notebook 
- **DuckDB** holds the data in a single local file. It reads CSV/Parquet straight from
  HTTPS, AWS S3 and Google Cloud Storage, so loading is one SQL statement per source.
- **The skill** (`taxi-demo/taxi-duckdb/SKILL.md`) is the only "integration": a short
  Markdown file with a `description` which the agent matches against the question,
  then the file path, table semantics, rules of thumb, and SQL templates.
- **No MCP server, no custom tools.** The agent runs ordinary Python against the file.

## Contents

```
taxi-demo/
├── duckdb_taxi_demo.ipynb            # 1. build the database from cloud sources
├── taxi-duckdb/SKILL.md              # 2. the skill the agent reads
├── load_opencode_taxi_skill.ipynb    # 3. install the skill for OpenCode + BioRouter
├── taxi_analysis.ipynb               # 4a. agent-generated: static analysis (matplotlib)
└── taxi_analysis_interactive.ipynb   # 4b. agent-generated: interactive rebuild (Altair/Vega-Lite)
```

## Try it

On a JupyterHub pod with `duckdb`, `jupyter-ai` and an ACP persona (OpenCode):

1. Run `duckdb_taxi_demo.ipynb` top to bottom.
2. Upload the `taxi-duckdb` folder to your home directory and run
   `load_opencode_taxi_skill.ipynb`.
3. Close the loading notebook's kernel (DuckDB locks the file while open).
4. In the Jupyter AI panel, pick the OpenCode persona and ask:
   - *What's the busiest hour of the day for taxi pickups?*
   - *Top 5 pickup neighbourhoods?* A JOIN to `zones` in its SQL proves it read the skill
   - *Average tip percentage by passenger count?* 
   - *How many trips cost more than $25, $50, $100 and $200?*
   - *Write all of this up as a notebook with charts.*
   