# Review and commit workflow

1. Run `git status` and `git remote -v`. Confirm origin is your Vishwateja-123 repository, not the reference repository or freeCodeCamp upstream.
2. Start from an up-to-date clean main branch and create a focused feature branch. Keep original material and history; do not force-push for organization changes.
3. Make one logical change. Use numbered Python/SQL files for answers and Markdown plus diagrams for design work.
4. Run the relevant tests and examples. Inspect diagrams, Markdown links, SQL assumptions and expected outputs.
5. Stage specific paths. Review `git diff --cached --find-renames`, `git diff --cached --stat` and `git diff --cached --check`.
6. Resolve uncertain requirement mappings before submitting. Use a descriptive commit such as `feat: complete week-1 reference tasks with tests`.
7. Push a review branch and open a pull request. Review before merging; do not change old commits to hide or backdate new work.
8. After merging, check the default-branch links and reconcile only your own tracker row. File count and test count are not instructor acceptance.

Preserved training files should remain byte-identical during cleanup. Fix old snippets in a separate commit with tests and an explanation.
