# KubeHealing

A Hugo blog about Kubernetes troubleshooting, CKA exam prep, and cloud-native observability, built on [PaperMod](https://github.com/adityatelange/hugo-PaperMod) and deployed to GitHub Pages.

## Running locally

Install Hugo (extended edition, required for the theme's SCSS pipeline):

```bash
# macOS
brew install hugo

# confirm you have the "extended" build
hugo version
```

Clone the repo with submodules (the theme is a git submodule):

```bash
git clone --recurse-submodules https://github.com/SenudaJK/kubehealing.git
cd kubehealing
```

If you already cloned without `--recurse-submodules`:

```bash
git submodule update --init --recursive
```

Start the dev server with live reload (includes drafts):

```bash
hugo server --buildDrafts --buildFuture
```

Visit `http://localhost:1313/`.

## Adding a new post

```bash
hugo new content posts/my-new-post.md
```

This uses the archetype at `archetypes/posts.md`, which pre-fills frontmatter. Edit the file and set:

```yaml
title: "Your Title"
date: 2026-01-01
description: "One or two sentences for SEO and social previews."
tags: ["kubernetes", "..."]
categories: ["troubleshooting"]   # one of: troubleshooting, cka-prep, observability, tutorials
draft: false                       # flip to false to publish
```

To insert the in-content ad placeholder after a heading in the post body, add the shortcode:

```markdown
{{< ad-in-content >}}
```

(See the existing posts in `content/posts/` for placement examples — it's used after the second `##` heading.)

## Ad placeholders

Three ad slots are wired into the templates as empty, clearly labeled `<div>` placeholders (no AdSense script is included yet):

| Slot | Location | File |
|---|---|---|
| Leaderboard | Below the homepage hero | `layouts/_partials/ad-leaderboard.html` |
| Sidebar (300x250) | Homepage + post pages | `layouts/_partials/ad-sidebar.html` |
| In-content | Manually placed via shortcode in post body | `layouts/_shortcodes/ad-in-content.html` |

Once your AdSense account is approved, drop your `<ins class="adsbygoogle">` snippet into each of these files, and add the AdSense loader `<script>` tag to `layouts/_partials/extend_head.html` (create this file if it doesn't exist — PaperMod calls it automatically from `head.html`).

## Deployment

`.github/workflows/hugo.yml` builds and deploys the site to GitHub Pages automatically on every push to `main`, using GitHub's official `actions/deploy-pages` flow (no `gh-pages` branch needed).

### One-time GitHub Pages setup

1. Push this repo to GitHub (see next section).
2. In the repo, go to **Settings → Pages**.
3. Under **Build and deployment → Source**, select **GitHub Actions**.
4. Push to `main` (or re-run the workflow from the **Actions** tab) — the site will build and deploy automatically.

## Next steps

### (a) Push this to your GitHub repo

```bash
cd /Users/senudajayathilaka/kubehealing
git add -A
git commit -m "Initial Hugo site: PaperMod theme, ad slots, sample posts, GH Pages workflow"
git branch -M main
git remote add origin https://github.com/SenudaJK/kubehealing.git
git push -u origin main
```

Then in the GitHub repo: **Settings → Pages → Source → GitHub Actions**. The workflow will run on push and publish to `https://SenudaJK.github.io/kubehealing/`.

> The `hugo.toml` `baseURL` is already set to `https://SenudaJK.github.io/kubehealing/` to match this repo name. If you rename the repo, update `baseURL` to match.

### (b) Connect a custom domain

1. Buy/point your domain's DNS at GitHub Pages:
   - **Apex domain** (`example.com`): add `A` records pointing to GitHub Pages' IPs (`185.199.108.153`, `.109.153`, `.110.153`, `.111.153`), or an `ALIAS`/`ANAME` record if your DNS provider supports one.
   - **Subdomain** (`blog.example.com`): add a `CNAME` record pointing to `SenudaJK.github.io`.
2. In the GitHub repo: **Settings → Pages → Custom domain**, enter your domain, save. GitHub will create a `CNAME` file in the deployed site automatically (or add one to `static/CNAME` in this repo so it survives rebuilds).
3. Update `baseURL` in `hugo.toml` to your custom domain, e.g. `baseURL = "https://blog.example.com/"`.
4. Wait for DNS propagation, then enable **Enforce HTTPS** in the same Pages settings once GitHub issues the certificate.

## Notes

- Category taxonomy pages (`/categories/troubleshooting/`, `/categories/cka-prep/`, `/categories/observability/`, `/categories/tutorials/`) only exist once at least one published post uses that category. The three sample posts cover `troubleshooting`, `tutorials`, and `observability` — the `cka-prep` menu link will 404 until you publish a post in that category.
- `content/privacy-policy.md`, `content/terms.md`, `content/contact.md`, and `content/about.md` contain placeholder copy required for AdSense review — edit them with real details before applying.
- Add a real `static/favicon.ico` and `static/images/og-default.png`, then uncomment the corresponding lines in `hugo.toml` (`params.assets.favicon`, `params.images`).
