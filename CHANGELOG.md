# Changelog

## 0.0.4

- Format Lisp runtime diagnostics without relying on inherited exception
  fields, preserving source locations and stack frames across runtimes.
- Skip nested `zuzu-js` command-runner checks where child arguments are not
  propagated by `Proc.run`.

## 0.0.3

- Mark distribution stable.

## 0.0.2

- Keep author-only runner under tests and emit valid skipped TAP when not
  author testing.

## 0.0.1

- Initial trial release.
