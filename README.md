# Silent Witness Agent Skills

Agent skills for integrating with the [Silent Witness API](https://docs.silentwitness.ai). Use these with Claude Code, Cursor, OpenAI Codex, and other AI coding tools to get accurate, context-aware integration code.

## Available Skills

| Skill | Description |
|-------|-------------|
| [sw-rest-api](skills/sw-rest-api/SKILL.md) | REST API reference — endpoints, field names, request/response formats, analysis types, async polling |
| [sw-sdk](skills/sw-sdk/SKILL.md) | Go & TypeScript SDK reference — method signatures, `StartAnalysis` helper, field naming, error handling |

## Quick Install

### Claude Code

```bash
mkdir -p .claude/skills/sw-rest-api .claude/skills/sw-sdk

curl -o .claude/skills/sw-rest-api/SKILL.md \
  https://docs.silentwitness.ai/skills/sw-rest-api.md

curl -o .claude/skills/sw-sdk/SKILL.md \
  https://docs.silentwitness.ai/skills/sw-sdk.md
```

Then invoke `/sw-rest-api` or `/sw-sdk` in Claude Code.

### Cursor

```bash
mkdir -p .cursor/rules

curl -o .cursor/rules/sw-rest-api.md \
  https://docs.silentwitness.ai/skills/sw-rest-api.md

curl -o .cursor/rules/sw-sdk.md \
  https://docs.silentwitness.ai/skills/sw-sdk.md
```

### OpenAI Codex

```bash
curl -o AGENTS.md \
  https://docs.silentwitness.ai/skills/sw-rest-api.md

# To include both skills:
echo -e "\n---\n" >> AGENTS.md
curl https://docs.silentwitness.ai/skills/sw-sdk.md >> AGENTS.md
```

## Documentation

- [Build with LLMs](https://docs.silentwitness.ai/guides/building-with-llms) — Full setup guide
- [API Reference](https://docs.silentwitness.ai/api-reference/overview) — REST API docs
- [SDK Quickstart](https://docs.silentwitness.ai/sdks/quickstart) — Get started with Go or TypeScript
- [Analysis Types](https://docs.silentwitness.ai/guides/analysis-types) — Choose the right analysis type

## Environments

| Environment | API | Dashboard |
|-------------|-----|-----------|
| Production | `https://api.silentwitness.ai/api` | [app.silentwitness.ai](https://app.silentwitness.ai) |
| Staging | `https://api.staging.silentwitness.ai/api` | [app.staging.silentwitness.ai](https://app.staging.silentwitness.ai) |

## License

MIT
