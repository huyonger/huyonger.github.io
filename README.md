# huyonger.github.io

Source of my personal homepage: <https://huyonger.github.io/>

Built with [AcademicPages](https://github.com/academicpages/academicpages.github.io) (MIT license, a fork of Minimal Mistakes).
The GitHub Actions workflow in `.github/workflows/jekyll.yml` builds and deploys the site on every push to `master`.

| What | Where |
|---|---|
| Sidebar: name, title, location, links | `_config.yml` → `author` |
| Page content: about, news, projects, publications, education, experience | `_pages/about.md` |
| Blog posts (listed at /blog/, linked from the header) | `_posts/YYYY-MM-DD-title.md`; start from `_drafts/post-template.md` |
| Header links | `_data/navigation.yml` |
| Layout and extra styles (page widths, masthead, news dates, venue badges); light theme only, no footer | bottom of `assets/css/main.scss`, `_layouts/default.html` |
| Photo and favicons | `images/` |

## Local preview

You need Ruby + Bundler, or Docker.

```bash
npm install && npm run build                 # builds assets/js/main.min.js (not committed)
bundle install
bundle exec jekyll serve -l -H localhost     # http://localhost:4000
# or: docker compose up
```
