# Contributing

Thanks for improving this stack.

- **Issues and PRs** are welcome for documentation fixes, safer defaults, and small configuration improvements.
- **Keep changes focused** — prefer a small, reviewable diff over drive-by refactors.
- **Validate Compose** before opening a PR:

```bash
docker compose config
```

- **Dashboards**: large JSON exports are hard to review; if you replace a dashboard, note the Grafana version you exported from in the PR description.

## Releases

When you are ready for a stable snapshot, create an annotated tag from `main`:

```bash
git tag -a v1.0.0 -m "First stable OSS release"
git push origin v1.0.0
```

Adjust the version to match your changelog or preferences.
