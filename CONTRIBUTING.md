# Contributing

Thank you for your interest in contributing to cloud-itonami-isic-0910!

## How to Contribute

1. **Report issues**: Open a GitHub issue if you encounter a bug or have a feature request.
2. **Submit PRs**: Fork the repository, create a feature branch, and submit a pull request with a clear description of your changes.
3. **Discuss design**: For major changes, please open an issue for discussion before implementing.

## Development

```bash
# Run tests
clojure -M:test

# Build
clojure -X:build
```

All source files use `.cljc` (portable Clojure) — no JVM-only constructs without a compelling reason (and documented in the PR).

## Scope Reminder

This actor is for CONTRACTOR field operations (dispatch, logistics, safety logging).
It is NOT for:
- Drilling decisions
- Well-control operations
- Hazmat handling authorization
- Operator-exclusive subsurface work

PRs that attempt to add operator-class operations will not be merged.

## License

By submitting contributions, you agree to license your work under AGPL-3.0-or-later.
