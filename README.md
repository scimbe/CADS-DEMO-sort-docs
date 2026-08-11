# CADS-DEMO-sort-docs

Documentation for [CADS-DEMO-sort](https://github.com/scimbe/CADS-DEMO-sort) — "Sort Arena". Jekyll
+ [Diátaxis](https://diataxis.fr), matching the same structure and hermetic-build process as
[CADS-Tunnel-docs](https://github.com/scimbe/CADS-Tunnel-docs) and
[CADS-devsystem-docs](https://github.com/scimbe/CADS-devsystem-docs).

Build locally:

```
docker run --rm -v "$PWD":/srv/jekyll -e JEKYLL_UID=$(id -u) -e JEKYLL_GID=$(id -g) \
  jekyll/jekyll:4 bash -c 'bundle install && bundle exec jekyll build --trace'
```

## Process

Every procedural claim (screenshots, described flows) is verified against the real, live
deployment at [sort.bunsenbrenner.org](https://sort.bunsenbrenner.org/) before being written down —
see [`_tutorials/first-participant.md`](_tutorials/first-participant.md).
