---
type: pattern
date: "2026-03-27"
source: A course participant's home-agent system (the memory pipeline)
tags:
  - testing
  - quality
  - scripts
---

# Smoke Tests with Real Data

Write smoke tests that use realistic sample data and import actual modules, not mocks. Test the real code path end-to-end.

## The Pattern

Instead of mocking dependencies, create a temp directory and run the real functions:

```python
SAMPLE_TRANSCRIPT = (
    "I just got off the Northwind diligence call and their routing layer "
    "is really interesting. They're doing dynamic model selection..."
)

class TestRawNoteCreation:
    def test_creates_markdown_file(self, tmp_vault):
        path = write_thought_note(
            tmp_vault, "2026-03-24-northwind-routing.md",
            SAMPLE_FRONTMATTER, SAMPLE_TRANSCRIPT,
        )
        assert path.exists()
        assert path.suffix == ".md"

    def test_frontmatter_parses_as_valid_yaml(self, tmp_vault):
        path = write_thought_note(...)
        post = frontmatter.load(str(path))
        assert isinstance(post.metadata, dict)
```

Key principles:
- **Realistic sample data** — a voice memo about a DD call, not `"test test test"`
- **Real module imports** — `from pipeline.vault import write_thought_note`, not mocks
- **Temp directories** — `tmp_vault` fixture creates a clean vault structure, torn down after
- **Round-trip validation** — write a note, process it, read it back, verify it parses

## Why It Works

- Tests the actual code path, so refactoring is safe
- Realistic data catches edge cases that synthetic data misses (special characters, long text, date formats)
- No mock maintenance — when the real code changes, the test either passes or fails, it doesn't silently pass because the mock is stale

## When to Use

Any pipeline that transforms data (voice → text → structured notes, CSV → database, API → local files). Start with one smoke test that runs the full pipeline on a sample input and checks the output format. That one test gives you confidence to refactor.

## Source

Derived from `tests/test_smoke.py` (205 lines, 15 tests in 2 classes). Tests both raw note creation and post-processing, validating frontmatter, body content, wikilinks, and round-trip parsing.
