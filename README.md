# racecar docs

Documentation site for the MAC Formula Electric racecar firmware. This repository
contains only the MkDocs site, extracted from the `docs/` folder of the
[`macformula/racecar`](https://github.com/macformula/racecar) monorepo so it can
be built and deployed independently on Vercel.

## Local preview

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
mkdocs serve
```

Open <http://127.0.0.1:8000>.

## Build

```bash
mkdocs build
```

The static site is written to `site/`.

## Vercel

`vercel.json` defines the build:

- install `requirements.txt`
- run `mkdocs build`
- serve `site/`

Import this repository in Vercel and deploy. No environment variables are
required.

## Updating the docs

The source of truth for the site content is this repository. The upstream
firmware lives at <https://github.com/macformula/racecar>.
