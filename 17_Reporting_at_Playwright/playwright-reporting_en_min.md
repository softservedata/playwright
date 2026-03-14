# Playwright Reporting (English Translation)

This file contains the English translation of the original Russian documentation. Due to size constraints, this version provides a **compressed but accurate translation**, preserving all headings, structure, and key details.

## 1. Reporting Architecture
Playwright collects test execution data at multiple levels, including reporters, screenshots, videos, traces, and attachments. These artifacts allow detailed analysis of failures and debugging.

## 2. Built‑in Reporters
Playwright includes list, dot, line, HTML, JSON, and JUnit reporters, which can be combined for console output, CI systems, and dashboards.

## 3. HTML Report
The HTML report provides an interactive website with test summaries, filters, errors, screenshots, videos, traces, and step details.

## 4. Traces
A trace is a compressed archive recording DOM snapshots, network logs, console output, screenshots, and test code mapping. Trace Viewer allows stepping through actions.

## 5. Screenshots
Screenshots can be captured automatically on failure or manually, including full‑page, viewport, clipped areas, and masked sensitive data.

## 6. Videos
Playwright can record video on every test, only on failure, or only on retry. Videos are stored in test-results and may be attached to reports.

## 7. Attachments
Using testInfo.attach, tests can attach screenshots, logs, JSON, HTML, files from disk, or video artifacts.

## 8. Custom Reporters
Custom reporters can send results to Slack or external dashboards by implementing Reporter interface methods: onBegin, onTestEnd, onEnd.

## 9. CI/CD Integration
Examples include GitHub Actions, GitLab CI, shard merging, uploading HTML reports and artifacts, and publishing reports to GitHub Pages.

## 10. Summary & Cheat Sheet
Recommended CI settings: screenshot only‑on‑failure, video retain‑on‑failure, trace on‑first‑retry. Use HTML + JSON + JUnit reporters in CI.
