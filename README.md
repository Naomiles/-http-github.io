# Naomiles — Personal Website

Personal website built with [Jekyll](https://jekyllrb.com) and the
[Chirpy](https://github.com/cotes2020/jekyll-theme-chirpy) theme, deployed on
[GitHub Pages](https://pages.github.com).

## Sections

| Section  | Where                                   | How to update                                   |
| -------- | --------------------------------------- | ----------------------------------------------- |
| Blog     | `_posts/`                               | Add a file named `YYYY-MM-DD-title.md`          |
| Skills   | `_skills/` + `_data/`                   | Add a skill entry in `_skills/`                 |
| CV       | `_data/cv.yml`                          | Edit the structured CV data                     |
| About    | `_tabs/about.md`                        | Edit the markdown content                       |

## Local development

Requires Ruby (with DevKit). Then:

```console
$ bundle install
$ bundle exec jekyll serve
```

Open <http://127.0.0.1:4000/-http-github.io/> to preview the site.

## Deployment

Push to the `main` branch — the included
[GitHub Actions workflow](.github/workflows/pages-deploy.yml) builds and deploys
the site automatically. Make sure **Settings → Pages → Source** is set to
**GitHub Actions**.

## Customization checklist

- [ ] `_config.yml` — change `title`, `tagline`, `description`, `social.name`, `social.email`
- [ ] `_data/cv.yml` — fill in your real CV data
- [ ] `_skills/` — replace the sample skills with yours
- [ ] `_tabs/about.md` — write your own introduction
- [ ] `avatar` in `_config.yml` — add your profile picture
- [ ] Comments — configure `giscus` / `utterances` / `disqus` in `_config.yml` if wanted

> Note: the site is deployed as a *project page*, so `baseurl` is set to
> `/-http-github.io`. If you rename the repository to `<username>.github.io`,
> set `baseurl: ""` in `_config.yml` and update the local preview URL.
