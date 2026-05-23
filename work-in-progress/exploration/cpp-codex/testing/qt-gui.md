# Qt / GUI testing

Overall disposition: PYTHON_SPECIFIC_VARIANT.

Rewrite as `gui-testing.md`. The C++ Qt mechanics do not port cleanly, but the
testing concerns do: the test owns the application lifetime, waits for GUI
conditions, drives input through the framework, observes signals/events, and
keeps object lifetime deterministic.

## The test binary owns the QApplication

Disposition: PORTS_WITH_ADAPTATION.

For PySide/PyQt, use pytest-qt's `qtbot` fixture or a session fixture that
owns the application. Do not construct and destroy the application repeatedly.

```python
def test_button_click_emits_signal(qtbot: QtBot) -> None:
    button = QPushButton("Submit")
    qtbot.addWidget(button)
```

## CMake for a GUI test

Disposition: DROP.

No Python analogue. Replace with dependency and pytest marker guidance.

## Headless CI: the offscreen platform

Disposition: PORTS_WITH_ADAPTATION.

Keep the concept. Use `QT_QPA_PLATFORM=offscreen` or xvfb when needed. For web
UIs, use Playwright's headless browser instead of Qt guidance.

## Showing widgets and waiting for them

Disposition: PORTS_WITH_ADAPTATION.

Use `qtbot.addWidget`, `widget.show()`, and wait helpers.

```python
widget.show()
qtbot.waitExposed(widget)
```

## Pumping the event loop

Disposition: PORTS_WITH_ADAPTATION.

Use `qtbot.wait`, `qtbot.waitUntil`, and signal wait helpers instead of raw
sleep.

```python
qtbot.waitUntil(lambda: model.rowCount() == 1, timeout=1000)
```

## Simulated input

Disposition: PORTS_WITH_ADAPTATION.

Use framework input helpers.

```python
qtbot.mouseClick(button, Qt.MouseButton.LeftButton)
```

## Observing signals: QSignalSpy

Disposition: PORTS_WITH_ADAPTATION.

pytest-qt offers signal wait helpers; PySide/PyQt also expose `QSignalSpy`.

```python
with qtbot.waitSignal(form.submitted, timeout=1000) as signal:
    qtbot.mouseClick(button, Qt.MouseButton.LeftButton)

assert signal.args == [expected_user_id]
```

## Object lifetime

Disposition: PORTS_WITH_ADAPTATION.

Keep it. Let `qtbot` own widgets, avoid mixed ownership, and pump the event
loop when testing deferred deletion.

## Model/View testing

Disposition: PORTS_WITH_ADAPTATION.

Keep model invariant and behavior layers where Python Qt bindings expose the
same tester. Otherwise write explicit role, row, signal, and mutation tests.

## High-DPI

Disposition: PORTS_AS_IS.

Keep it. Use widget-relative geometry, not hard-coded pixels.
