# Artem Demchyshyn Blog

A minimal technical blog for Java backend engineering notes, built with Hugo and the PaperMod theme.

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

1. Push this repository to GitHub.
2. In the GitHub repository, open `Settings` -> `Pages`.
3. Set `Source` to `GitHub Actions`.
4. Push to `main`.

The workflow in `.github/workflows/hugo.yaml` builds the site with Hugo and deploys `public/` to GitHub Pages. The workflow sets the production `baseURL` from GitHub Pages automatically, so it works for both user sites and project sites.

## Analytics

The blog is prepared for GoatCounter, a lightweight privacy-friendly analytics service with a free hosted option for personal/public sites.

1. Create a site on [GoatCounter](https://www.goatcounter.com/).
2. Choose a site code, for example `artem-blog`.
3. Update `hugo.yaml`:

```yaml
params:
  analytics:
    goatcounter:
      enabled: true
      code: "artem-blog"
      enableInDevelopment: false
```

After deployment, stats will be available at:

```text
https://artem-blog.goatcounter.com/
```

Keep `enableInDevelopment: false` to avoid counting local visits from `hugo server`.

## Updating PaperMod

```powershell
git submodule update --remote --merge themes/PaperMod
```

Then test locally and commit the updated submodule pointer.
