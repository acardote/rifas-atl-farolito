# D-0003: Keep serving the site from the bucket's direct S3 REST endpoint, with no CDN and no cache invalidation

Status: proposed
Current: The site is served from the bucket's direct S3 REST endpoint at https://rifas-atl-farolito.s3.eu-west-1.amazonaws.com/index.html through the bucket policy's public `s3:GetObject`, with no CloudFront, no S3 website hosting and no cache invalidation step; CloudFront with OAC is deferred to a later parent.

## Context

Read-only reads on 2026-11-24 (listed in the linked parent's Move 1 evidence) showed that the bucket `rifas-atl-farolito` (eu-west-1) has no website configuration, a bucket policy that allows public `s3:GetObject` on every key and no `s3:ListBucket`, and no CloudFront distribution in the account that references it. The owner stated that the URL people were given is https://rifas-atl-farolito.s3.eu-west-1.amazonaws.com/index.html and that the site is dormant.

So the site is served today by the S3 REST endpoint, over HTTPS on the amazonaws.com host. A GET of the bucket root does not return the page, because listing is not granted and REST has no index document. A proxy or CDN outside this account cannot be seen from the account. Only the owner's statement covers it.

Options considered:

1. **Keep the direct S3 REST endpoint.** No infrastructure change, and the shared URL keeps working. A new object is served on the next request, because there is no cache layer to invalidate. Limits: HTTPS only on the amazonaws.com host, not on an own domain. No custom response headers (CSP, HSTS, `X-Content-Type-Options`). The bucket must stay publicly readable. Objects are served with their stored metadata only.
2. **CloudFront with Origin Access Control on the REST endpoint.** Gives HTTPS on an own domain (ACM certificate, DNS), response-header policies, a private bucket and edge caching. It needs a distribution, OAC, a bucket-policy change, a certificate, DNS, a default root object and invalidation on every deploy. It also gives the site a new URL, so links already shared to the S3 URL would stop showing the new site unless the public read stays during a transition.
3. **S3 static website hosting.** Adds an index document, so the root URL would serve the page. The website endpoint is HTTP only, it needs a bucket-configuration change, and it would change the URL. It gains nothing the shared URL needs.
4. **CloudFront in front of the website endpoint.** HTTPS at the edge, but the bucket must stay public, and it carries the costs of both 2 and 3.

## Decision

Option 1. Keep the direct S3 REST endpoint as the delivery path, and do not add a cache-invalidation step to the deploy.

- Option 2 (CloudFront + OAC) is **deferred**, not rejected. Its benefits (an own domain over HTTPS, response headers, a private bucket) are real, but it is infrastructure work with its own risks and a URL change, and the site is dormant with placeholder content. It belongs in a later parent, for example alongside the real-content update.
- Option 3 (website hosting) is **rejected**: HTTP only, a configuration change and a new URL, for no gain on the shared URL.
- Option 4 is **rejected**: it keeps the public bucket and adds the costs of both 2 and 3.

## Consequences

- A deploy is an object replacement only. Verification is a fetch of the shared URL and a byte comparison. There is no invalidation step and no CloudFront write permission is needed. The agent read-only identity holds `cloudfront:ListDistributions` only to re-check that no distribution references the bucket.
- Public `s3:GetObject` on the bucket is load-bearing. Removing or narrowing it takes the site down. Hardening it requires the deferred CloudFront + OAC move first.
- Security headers cannot be set. The page's CSP `<meta>` tag (D-0001) is the only browser-side policy. HSTS and `X-Content-Type-Options` are absent.
- Correct Content-Type and Cache-Control on each object are the deploy's job (D-0002), because the endpoint serves them as stored. Browsers and intermediaries may still cache. `Cache-Control: no-cache` on `index.html` limits that.
- The bucket root URL keeps returning an error rather than the page. Sharing the root URL is not supported.
- If the owner later wants an own domain or response headers, this record is superseded by a record for the CloudFront + OAC move.

## Links

Linked: #1
