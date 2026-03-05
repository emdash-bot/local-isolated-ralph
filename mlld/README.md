# Smithers → mlld

This directory is an mlld rebuild of the Smithers workflow examples in this repo.

## What's here

| File | Original | Description |
|---|---|---|
| `hello-world.mld` | `examples/hello-world-local/workflow.ts` | Single LLM task: generate a hello-world HTML page |
| `counter-app.mld` | `examples/counter-local/workflow.tsx` | 4-step workflow: optional VCS setup, LLM implementation, optional jj commit/push |

---

## Key differences: Smithers vs mlld

### Smithers

```ts
const { smithers, outputs } = createSmithers({ page: z.object({...}) }, { dbPath: "..." });
const agent = new CodexAgent({ sandbox: "workspace-write" });

export default smithers(() =>
  Workflow({
    name: "hello-world-agent",
    children: Task({ id: "generate", output: outputs.page, agent, children: `...prompt...` }),
  }),
);
```

- TypeScript/Bun, JSX-style composition
- Zod schema validation per task output
- `skipIf` for conditional tasks
- SQLite-backed state for resumability
- Each workflow is a Node module

### mlld equivalent

```mlld
exe llm @generate_page = sh {
  codex exec --full-auto --json "...prompt..."
}

var @raw    = @generate_page()
var @result = @raw | @parse.llm

show `Generated: @result.filePath`
```

- Declarative scripting language, no build step
- `llm` label = automatic checkpointing (analogous to Smithers' SQLite state)
- `| @parse.llm` extracts JSON from LLM response
- `checkpoint "name"` = named resume points
- `if / else` replaces `skipIf`
- Parallel tasks via `for parallel(N)`

### Mapping

| Smithers | mlld |
|---|---|
| `createSmithers({ outputs })` | `var @result = ... \| @parse.llm` |
| `new CodexAgent({ ... })` | `exe llm @task = sh { codex exec ... }` |
| `Workflow > Task[]` | sequential script flow |
| `Task.skipIf` | `if @condition [...]` |
| SQLite state / resumability | `checkpoint` + `--resume @task` |
| `z.object({ ... })` validation | `@parse.llm` + manual checks |
| `for parallel(N)` (Smithers) | `for parallel(N) @x in @items` |

---

## Usage

### hello-world

```bash
# Run from repo root
mlld run mlld/hello-world.mld

# Re-run fresh (clear checkpoint)
mlld run mlld/hello-world.mld --fresh

# Resume after interruption
mlld run mlld/hello-world.mld --resume @generate_page
```

### counter-app

```bash
# Basic run (creates react-counter-app/ in cwd)
mlld run mlld/counter-app.mld

# Custom app name
SMITHERS_APP_NAME=my-counter mlld run mlld/counter-app.mld

# With jj VCS
SMITHERS_JJ_REPO=https://... SMITHERS_JJ_BOOKMARK=main mlld run mlld/counter-app.mld

# With custom model
SMITHERS_MODEL=gpt-4o mlld run mlld/counter-app.mld

# Resume after interruption (skips LLM re-call)
mlld run mlld/counter-app.mld --resume @implement_app
```

---

## Design notes

**Why mlld is a good fit for Smithers patterns:**

1. **Checkpointing is free.** Any `exe llm @fn` is automatically cached. Smithers needs an explicit SQLite DB path; mlld handles it transparently in `.mlld/checkpoints/`.

2. **Less ceremony.** No `createSmithers`, no `CodexAgent` constructor, no Zod schema registry. You declare what you need and move.

3. **Conditional flow is native.** `if @jj_repo [...] else [...]` is cleaner than `skipIf={!jjRepo}` sprinkled on every Task.

4. **Shell-native.** The VCS steps (`jj git clone`, `jj commit`) live naturally in `sh {}` blocks. In Smithers they're TypeScript async functions using Bun's `$` tagged template.

5. **Parallelism.** `for parallel(N)` handles the multi-reviewer benchmark pattern from `benchmark/smithers-benchmark.tsx` without any explicit Promise.all machinery.

**What Smithers does better:**

- Zod schemas catch output shape errors at task boundaries — mlld's `@parse.llm` is best-effort
- The JSX tree model is more composable for deeply nested or branching workflows
- TypeScript gives you type safety throughout
