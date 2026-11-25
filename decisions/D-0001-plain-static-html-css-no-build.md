# D-0001: Build the site as plain static HTML and CSS with no build step

Status: proposed
Current: The site is hand-written HTML and CSS in `site/`, with self-hosted assets (favicon included) referenced by relative URLs, a restrictive CSP `<meta>` tag, no framework, package manager or build step, and no third-party runtime requests.

## Context

The previous site is one `index.html` with inline CSS, no JavaScript, and 7 image files (about 336K). It loads three images from a third-party host (`via.placeholder.com`). The new site has the same shape (one page, a list of prizes, a date, logos), and its content is placeholders for now.

The deliverable is objects in a public S3 bucket, served directly by the S3 REST endpoint (D-0003). Two facts about that endpoint shape this decision:

- There is no way to set HTTP response headers per site (no CSP, HSTS or `X-Content-Type-Options` header) without a CDN in front. Any policy the page enforces must be in the page itself.
- The same object is reachable at a virtual-hosted URL (`https://<bucket>.s3.<region>.amazonaws.com/index.html`) and a path-style URL (`https://s3.<region>.amazonaws.com/<bucket>/index.html`). A root-relative URL such as `/assets/x.png` resolves to the wrong place under the path-style form. The bucket root does not serve `index.html`.

The repository is public and the bucket is public, so every committed or uploaded file is world-readable.

Options considered:

1. Plain HTML + CSS (optionally a little vanilla JS), no build. Nothing to install, the repo files are exactly what is uploaded, and reconciliation is a byte comparison.
2. A static-site generator or bundler (Astro, Vite, Eleventy, etc.). Templating and asset pipelines help multi-page sites and content managed as data. They add a package install and a build that runs third-party code, which is an agent hard stop and a supply-chain surface, for a single page.
3. A CSS framework from a CDN (Tailwind CDN, Bootstrap CDN). Quick to style, but adds third-party runtime requests and a dependency on an external host for a production page.

The deciding facts: a single page, placeholder content, a later content update that may change copy but not the page count, and the wish to reconcile deployed bytes against the repo exactly.

## Decision

Option 1, with these rules:

- The site lives in `site/` and only the files in `site/` are uploaded. No `package.json`, no build, no CDN assets.
- Every asset reference is relative with no leading slash (for example `assets/hero.png`), so the page works at both the virtual-hosted and the path-style URL. Links to the page itself name `index.html` explicitly.
- The page declares its favicon with a relative path, `<link rel="icon" href="assets/favicon.png">` (file name illustrative). Without it, browsers request `/favicon.ico` at the host root, which is outside the bucket under the path-style URL and an error under the virtual-hosted URL.
- CSS lives in a separate self-hosted file. There is no inline `<style>`, `style=` attribute or inline script.
- The page carries this CSP in a `<meta http-equiv="Content-Security-Policy">` tag: `default-src 'none'; img-src 'self'; style-src 'self'; font-src 'self'; base-uri 'none'; form-action 'none'`. If self-hosted JS is ever added, `script-src 'self'` is added with it and nothing wider.
- Fonts are system fonts or self-hosted with their license checked.
- Every committed image has its EXIF/GPS metadata stripped before commit, shown by a metadata dump of the committed file.

## Consequences

- Deploy and reconcile are simple: the uploaded files are the repo files at a named commit, so a hash comparison proves what is live.
- No dependency installs, so no agent hard stop at build time.
- The CSP makes the browser refuse requests to other hosts, so the "no third-party requests" rule is enforced, not only intended, with one limit: `'self'` means the page's origin. Under the virtual-hosted URL that is the bucket's own host. Under the path-style URL it is the shared S3 host `https://s3.eu-west-1.amazonaws.com`, which serves every bucket in the region, so there `'self'` also admits objects from other public buckets on that host. Relative URLs keep the page's own requests in this bucket either way. A `<meta>` CSP cannot express `frame-ancestors` and is no substitute for response headers; those need CloudFront (deferred in D-0003).
- Rules out templating and an image pipeline. Images are optimized and stripped by hand before commit. If the later content update makes the site multi-page or data-driven, this decision should be amended or superseded then, not worked around.
- Rules out Google Fonts and other CDN fonts at runtime.

## Links

Linked: #1
