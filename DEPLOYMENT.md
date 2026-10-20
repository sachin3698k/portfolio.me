# Deploying this site to Cloudflare

## TL;DR — what was actually breaking the build

Two independent problems, and the error message only hints at the second one:

1. **The entry file was named `Main.html`, not `index.html`.** Wrangler's
   automatic asset detection does not scan for "html, css and js" as the error
   message claims — it looks *specifically* for `index.html`, either at the
   project root or exactly one level down. A file called `Main.html` is
   invisible to it.
2. **There was no `wrangler.jsonc` / `wrangler.toml`**, so `npx wrangler deploy`
   fell back to auto-detection, which then threw:

   ```
   ✘ [ERROR] Could not detect a directory containing static files (e.g. html, css and js) for the project
   ```

Even if this had been set up as Cloudflare **Pages** instead of a Worker, the
site would still have been broken: Pages maps `/` to `index.html`, so
`Main.html` would have produced a 404 at the site root.

## 1. Was this set up as a Worker instead of Pages?

**Yes — it is configured as a Cloudflare Worker (Workers Builds), not Pages.**
That is the correct/intended modern setup, but it is what makes
`npx wrangler deploy` run its autoconfig path.

Evidence from the build log:

- `Could not detect a directory containing static files...` is emitted **only**
  by the static-assets autoconfiguration inside `wrangler deploy`. It is never
  produced by `wrangler pages deploy`.
- `No dependencies detected to cache.` means Workers Builds found no
  `package.json` / lockfile — expected here, since this is a plain HTML/CSS
  site and needs no build step. This line is harmless.

Important: **the Worker-vs-Pages choice was not what broke the build.** The
missing `index.html` and the missing Wrangler config were. Switching to Pages
alone would not have fixed it.

## 2. Exact dashboard settings

In the Cloudflare dashboard for the Worker → **Settings → Build**, set:

| Setting | Value |
| --- | --- |
| Build command | *(leave empty)* — or `echo "no build step required"` if the UI insists |
| Deploy command | `npx wrangler deploy` |
| Root directory | `/` (repository root — the folder containing `wrangler.jsonc`) |
| Build output directory | *(not applicable to Workers — do not set; it lives in `wrangler.jsonc`)* |

There is no compile step. `npx wrangler deploy` is correct; it now reads
`wrangler.jsonc` instead of guessing.

If you deliberately want **Cloudflare Pages** instead, use a Pages project with:

| Setting | Value |
| --- | --- |
| Framework preset | None |
| Build command | *(empty)* |
| Build output directory | `public` |

…and do **not** use `npx wrangler deploy` there — Pages publishes the output
directory itself.

## 3. Required config files (now committed)

### `wrangler.jsonc`

```jsonc
{
  "name": "portfolio-me",
  "compatibility_date": "2026-10-20",
  "assets": {
    "directory": "./public"
  }
}
```

> **Change `name` to match your Worker's name in the Cloudflare dashboard.**
> If it differs, `wrangler deploy` will create a *new* Worker instead of
> updating the one you are looking at.

### Repository layout

```
portfolio.me/
├── public/
│   ├── index.html     (was Main.html)
│   └── design.css
├── wrangler.jsonc
├── .gitignore
└── README.md
```

`public/` is the assets directory. Keeping assets in their own folder matters:
pointing `"directory": "."` at the repo root also uploads `wrangler.jsonc`,
`README.md`, `.gitignore` and the `.wrangler/` cache to your website
(verified: 9 files uploaded vs 2 from `public/`).

### Why not just a `--assets` flag?

`npx wrangler deploy --assets ./public` does work, but it is not enough on its
own — verified against wrangler 4.151.3, it then fails with:

```
✘ [ERROR] You need to provide the name of your worker.
```

and, once a name is supplied:

```
✘ [ERROR] A compatibility_date is required when uploading a Worker.
```

So the config file is the clean fix. A one-liner deploy command would have to be
`npx wrangler deploy --assets ./public --name portfolio-me --compatibility-date 2026-10-20`,
which is more fragile than a committed `wrangler.jsonc`.

### Do NOT use `pages_build_output_dir`

A `wrangler.toml` containing `pages_build_output_dir = "./public"` does **not**
fix `npx wrangler deploy`. Verified output:

```
▲ [WARNING] It seems that you have run `wrangler deploy` on a Pages project,
            `wrangler pages deploy` should be used instead.
✘ [ERROR] Missing entry-point to Worker script or to assets directory
```

That key only applies to `wrangler pages deploy`.

## Verifying locally

```bash
npx wrangler deploy --dry-run      # must print: Read 2 files from .../public
```

## Note

`design.css` is not referenced by `index.html` — the page carries its own inline
`<style>` block. The file is kept and deployed, but nothing links to it. Add
`<link rel="stylesheet" href="/design.css">` to the `<head>` only if you intend
to switch to the external stylesheet (its `.topnav` colour differs slightly from
the inline one, so the header would change appearance).
