# huyonger.github.io

Source of my personal homepage: <https://huyonger.github.io/>

Built with [AcademicPages](https://github.com/academicpages/academicpages.github.io) (MIT license, a fork of Minimal Mistakes).
The GitHub Actions workflow in `.github/workflows/jekyll.yml` builds and deploys the site on every push to `master`.

| What | Where |
|---|---|
| Sidebar: name, title, location, links | `_config.yml` → `author` |
| Page content: about, news, publications, education, awards | `_pages/about.md` |
| Extra styles: news dates, venue badges, collapsible blocks | bottom of `assets/css/main.scss` |
| Photo and favicons | `images/` |

## Local preview

You need Ruby + Bundler, or Docker.

```bash
npm install && npm run build                 # builds assets/js/main.min.js (not committed)
bundle install
bundle exec jekyll serve -l -H localhost     # http://localhost:4000
# or: docker compose up
```
