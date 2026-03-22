# Component Contracts — Figma Skills

Contract-driven Figma component generation. JSON contracts → Primitive + Semantic variable collections → fully variable-bound component sets. Skills for Cursor + Figma MCP.

**Figma community file:** [figma.com/community/file/1617658204115347372](https://www.figma.com/community/file/1617658204115347372)

---

## What this is

A structured JSON contract file is the authoritative definition for a component. Everything downstream — Figma variables, component sets, CSS, native — is derived from it deterministically. The contract is source code. Figma is one build target.

This is spec-driven, not prompt-driven. The skills are compilers, not creative tools.

```
button.contract.json  →  cc-figma-tokens  →  Primitive + Semantic variables
                      →  cc-figma-component  →  Fully variable-bound component set
```

---

## Repo structure

```
skills/
  cc-figma-tokens/
    SKILL.md          — builds Primitive + Semantic variable collections from token files
  cc-figma-component/
    SKILL.md          — generates component sets from contracts
examples/
  contracts/
    button.contract.json
    accordion.contract.json
  tokens/
    primitive/
      color.tokens.json
    semantic/
      semantic.tokens.json
schema/
  schema.json         — JSON Schema for contract validation
.component-contracts.example
```

---

## Getting started

**1. Clone this repo**

```bash
git clone https://github.com/nvillapiano/component-contracts-figma
```

**2. Configure your environment**

```bash
cp .component-contracts.example .component-contracts
```

Edit `.component-contracts` and fill in your Figma access token, file key, and paths to your token and contract directories.

**3. Install the skills**

Copy both skill directories into your agent's skills directory:

```bash
cp -r skills/cc-figma-tokens ~/.cursor/plugins/local/figma-use/skills/
cp -r skills/cc-figma-component ~/.cursor/plugins/local/figma-use/skills/
```

**4. Define your tokens**

Start from `examples/tokens/`. Primitives are raw values, Semantic are aliases into Primitives. Replace the example values with your own system's design decisions.

**5. Write a contract**

Use `examples/contracts/button.contract.json` as a template. Define your component's tokens, props, states, variants, and composition slots. Validate against `schema/schema.json`.

**6. Build variable collections**

In Cursor with Composer 2 + MAX Mode:

```
Read .component-contracts. Load cc-figma-tokens, figma-use, and figma-generate-library skills.

Build the Primitive and Semantic variable collections from the token files.
Verify your Phase 0 plan, then run straight through to completion.
```

**7. Generate the component**

```
Read .component-contracts. Load cc-figma-component, cc-figma-tokens, figma-use, and figma-generate-library skills.

The Primitives and Semantic variable collections already exist — do not recreate them.

Build the [ComponentName] Figma component from the [name].contract.json contract.
Verify your Phase 0 plan, then run straight through to completion.
```

---

## Agent settings

- Use the most capable model available with maximum context. These skills require multi-step orchestration across many tool calls.
- In Cursor: use **Composer 2 with MAX Mode** enabled. Composer 2 Fast will work but expect significantly longer run times and occasional dropped instructions.
- The model resets to default on every new Cursor chat. Check your model selection before each run.

---

## Contracts and tokens can live anywhere

The file-based structure in this repo is a reference implementation. The same pipeline works from a CMS, a database, a Google Sheet, or any source that outputs the contract schema and W3C DTCG token format. The skills are readers — they don't care about the upstream.

---

## Troubleshooting

| Issue | Fix |
|-------|-----|
| Buttons rendering too tall | Frame sizing defaulting to FIXED. Re-run with updated skill — the HUG enforcement pass corrects it. |
| Variables showing as unresolved | Run `cc-figma-tokens` first. Semantic collection must exist before components are built. |
| Script too large for MCP | The skill handles chunking automatically via `new Function`. No action needed. |
| Generation Notes shows token gaps | Add missing aliases to `semantic.tokens.json` and re-run `cc-figma-tokens`. |
| Chevron rotation showing as −1.57° | Agent passed radians instead of degrees. Skill fix is in place — re-run the component. |

---

## Figma community file

The community file contains Button (72 variants) and Accordion (30 variants) generated entirely by this pipeline — fully variable-bound to Semantic tokens, with a `⚠️ Generation Notes` frame on each component page documenting every assumption and known limitation.

[View on Figma Community →](https://www.figma.com/community/file/1617658204115347372)

---

Built by [Nick Villapiano](https://www.linkedin.com/in/nicholasvillapiano/) at [One North](https://www.onenorth.com) · Generated with Figma MCP + Claude + Cursor
