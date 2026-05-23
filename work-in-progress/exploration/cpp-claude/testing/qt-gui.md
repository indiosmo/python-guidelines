# Analysis: cpp-testing-principles/qt-gui.md

Disposition: **DROP** the C++ Qt-specific content; **REPLACE** with a
short Python GUI / web-UI testing chapter if and only if the Python
codebase has a GUI surface to test.

## Rationale

The C++ chapter is specifically about Qt with Catch2: offscreen
platform, `QSignalSpy`, `QAbstractItemModelTester`, `QTest::qWait`,
event loops, high-DPI rendering. Almost none of this transfers:

- Python codebases rarely use Qt. The Python ports (`PySide6`,
  `PyQt6`) exist and have analogous `pytest-qt` infrastructure, but
  it is a niche choice.
- The factors2 codebase uses **Streamlit / FastAPI / Dagster** for
  user-facing surfaces -- no Qt at all.
- The web-UI testing story (Playwright / Selenium / requests-html) is
  a wholly different chapter that does not map to the C++ rules.

## Recommendation for the Python guide

1. **Drop the C++ chapter unmodified.** Do not write a Python "qt-gui.md"
   unless the codebase actually has a Qt surface.
2. **If the codebase has a Qt surface** (rare): the chapter ports
   directly using `pytest-qt`, which provides `qtbot.waitSignal`,
   `qtbot.waitUntil`, and `qtbot.mouseClick` -- direct analogues of
   `QSignalSpy::wait` and `QTest::qWaitFor`.
3. **If the codebase has a web UI** (much more common): write a new
   chapter on **Playwright** (the modern default for browser-driven
   tests) covering:
   - Page-object pattern (the Python equivalent of "factories +
     probes").
   - `expect(locator).to_be_visible()` waits (the equivalent of
     `qtbot.waitUntil`).
   - Snapshot/visual regression with `pytest-playwright-snapshot`.
   - Headless vs headed mode, and CI configuration.
4. **If the codebase has a FastAPI / Starlette surface**: use
   `httpx.AsyncClient` against a `TestClient`-mounted app; this is
   integration-test territory, covered by `philosophy.md`'s
   "integration tests" subsection.

## Suggested Python file: none by default

Skip the file. Add it only if the codebase has a GUI surface worth a
chapter. If the team wants a web-UI chapter, the natural name is
`python-testing-principles/web-ui.md` (Playwright-focused).
