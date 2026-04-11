# Git Commit Strategies

<!-- migrated-from: /Users/jasonr/.config/agent-docs/git-commit-strategies.md -->

## Atomic commits for interdependent changes

<!-- learned: 2026-02-15 -->

When a file rename and code that references the renamed file change together, they must be in a single commit. Splitting them would break bisectability — any intermediate commit where one change is present without the other would leave the project in a broken state. The rule of thumb: if change A doesn't make sense without change B (and vice versa), they belong in one commit regardless of how many files are touched.
