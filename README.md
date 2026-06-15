# Artem Demchyshyn Blog

A minimal technical blog for Java backend engineering notes, built with Hugo and the PaperMod theme.

**Live:** <https://demchaav.github.io/blog/>

## Stack

- Hugo
- PaperMod
- GitHub Pages
- GitHub Actions
- No Node.js required for this setup

## Project Structure

```text
.
|-- archetypes/
|-- content/
|   |-- about.md
|   |-- archives.md
|   |-- search.md
|   `-- posts/
|-- themes/
|   `-- PaperMod/
|-- .github/
|   `-- workflows/
|-- hugo.yaml
`-- README.md
```

## Local Development

Install Hugo Extended `0.146.0` or newer. PaperMod declares this as its minimum Hugo version.

Clone the repository with submodules, or initialize them after cloning:

```powershell
git submodule update --init --recursive
```

Run the site locally:

```powershell
hugo server -D
```

Open:

```text
http://localhost:1313/
```

Create a new post:

```powershell
hugo new posts/my-post-title.md
```

Set `draft: false` when the post is ready to publish.

## Publishing to GitHub Pages

The site is already live at <https://demchaav.github.io/blog/> and redeploys automatically on every push to `main`. The one-time setup (kept for reference):

1. Push this repository to GitHub.
2. In the GitHub repository, open `Settings` -> `Pages`.
3. Set `Source` to `GitHub Actions`.
4. Push to `main`.

The workflow in `.github/workflows/hugo.yaml` builds the site with Hugo and deploys `public/` to GitHub Pages. The workflow sets the production `baseURL` from GitHub Pages automatically, so it works for both user sites and project sites.

## Analytics

Analytics use [GoatCounter](https://www.goatcounter.com/) (privacy-friendly, no cookies). It is enabled in `hugo.yaml` with the site code `demchaav`:

```yaml
params:
  analytics:
    goatcounter:
      enabled: true
      code: "demchaav"
      enableInDevelopment: false
```

Stats: <https://demchaav.goatcounter.com/>. The script loads in production only, so local `hugo server` visits are not counted.

## Comments

Comments use [giscus](https://giscus.app/) backed by this repository's GitHub Discussions (Announcements category). The loader lives in `layouts/partials/comments.html` and is enabled via `params.comments: true`; the widget follows the site's light/dark theme.

## Updating PaperMod

```powershell
git submodule update --remote --merge themes/PaperMod
```

Then test locally and commit the updated submodule pointer.
