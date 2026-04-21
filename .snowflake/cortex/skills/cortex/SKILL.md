---
name: cortex
description: "Cortex AI patterns and prompt auditing for Snowflake AI_COMPLETE and AI_PARSE_DOCUMENT. Use for any Cortex AI work — calling, parsing, debugging, or auditing prompts. Triggers: AI_COMPLETE, cortex, AI_PARSE_DOCUMENT, prompt audit, cortex error, parse error, dollar-quoting"
---

# Cortex AI

## When to Use

When calling, parsing, debugging, or auditing Cortex AI functions (AI_COMPLETE, AI_PARSE_DOCUMENT).

## Intent Detection

| Intent | Triggers | Load |
|--------|----------|------|
| PATTERNS | AI_COMPLETE, cortex error, model not found, file not accessible, diagnostics, dollar-quoting, parse_cortex_json | `patterns/SKILL.md` |
| AUDIT | audit prompts, prompt quality, parse error, wrong keys, shallow explanation, prompt review | `prompt-audit/SKILL.md` |

## Workflow

```
User request
  ↓
Intent Detection
  ├─→ PATTERNS → Load patterns/SKILL.md
  └─→ AUDIT    → Load prompt-audit/SKILL.md
```

## Capabilities

- **Patterns**: AI_COMPLETE calling with dollar-quoting, response parsing (markdown fences, double-encoding), AI_PARSE_DOCUMENT, stage requirements, 5-step diagnostics
- **Prompt Audit**: 8-item checklist for JSON reliability, content completeness, doc_search pattern, injection safety

## Output

Correct Cortex AI integration or a prompt audit report with PASS/FAIL per item.
