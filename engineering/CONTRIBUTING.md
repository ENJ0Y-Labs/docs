# Contributing Guide

## Before Making Changes

1. Read the repository's `AGENTS.md` or `AGENT.md` if present and follow its instructions.
2. Read the relevant RetailOS PRD, requirements, user flow, and decision records in this repository.
3. Confirm the issue's scope and acceptance criteria before coding.

## During Development

- Keep changes focused on the agreed issue.
- Do not silently change product behavior; propose requirement changes for review.
- Add or update tests for behavior changes and bug fixes.
- Keep API and client behavior aligned.
- Update documentation when approved changes alter requirements or decisions.

## Before Opening a Pull Request

- Run the tests, lint checks, and build checks relevant to the changed codebase.
- Report exactly which checks passed and which were not run.
- Explain the change, its rationale, and any migration or compatibility implications.
- Link the issue and relevant requirement IDs.
- Do not include unrelated formatting or refactoring.

## Review Expectations

A pull request should be small enough to review, include appropriate tests, and satisfy the issue's acceptance criteria. Never claim a test passed unless it was actually run.
