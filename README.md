<p align="center"><img src="https://raw.githubusercontent.com/go-deltasync/brand/main/social/go-deltasync.png" alt="go-deltasync/docs" width="720"></p>

# go-deltasync/docs

Versioned documentation for the [go-deltasync](https://github.com/go-deltasync)
tools, built with [MkDocs Material](https://squidfunk.github.io/mkdocs-material/)
and versioned with [mike](https://github.com/jimporter/mike). Published to the
`gh-pages` branch and served at <https://go-deltasync.github.io/docs/>.

The organization landing page ([go-deltasync.github.io](https://go-deltasync.github.io))
links here.

## Local preview

```bash
python -m venv .venv && . .venv/bin/activate
pip install -r requirements.txt
mkdocs serve                       # http://localhost:8000 (current sources)
mike serve                         # preview the versioned site
```

## Releasing a new docs version

```bash
mike deploy --push --update-aliases <version> latest
mike set-default --push latest
```
