---
name: ship-with-privacy
description: >
  Generic workflow for publishing changes to a content/portfolio site with a
  mandatory privacy and secret scrub before every deploy. Covers the full
  pipeline: edit content → scrub (no secrets, internal addresses, or
  harvestable PII; blur third-party personal data in screenshots) → verify
  build → commit → deploy → confirm live. Adapt the repo path, git remote,
  and deploy command to your own stack.
---

<!-- Public, trimmed version shared from smereski.com. Adapt <placeholders> to your own repo, host, and deploy tooling. -->

# ship-with-privacy — publish with a mandatory privacy scrub

A reusable workflow for shipping changes to any public content or portfolio
site. The core discipline: **the privacy scrub is non-skippable**. Everything
else (edit → build → commit → deploy → verify) is standard, but the scrub
gate between editing and shipping is what makes publishing to a site under
your real name safe.

---

## 0. Pre-flight checklist

Run these in order. Do not skip to step 3 or 4.

1. Make the content or code change (section 1).
2. Run the **privacy scrub** (section 2) — BLOCKING.
3. Verify the build (section 3) — build must exit 0 and email literal must
   not appear in the output bundle.
4. Commit, push, deploy, confirm live (section 4).

---

## 1. Edit content

Content-driven sites keep data in arrays or flat files that feed templates.
Edit the data; the pages render themselves. Common patterns:

- **Project cards** — a `projects[]` array in a data file. Fields typically
  include: `id`, `title`, `tagline`, `description`, `tech[]`, `status`, `year`.
  Status should reflect reality (`live` / `beta` / `private`); read the actual
  repo before setting it — do not trust memory.
- **Blog posts** — a `posts[]` array, newest first. Fields: `slug`, `title`,
  `tagline`, `date`, `minutes`, `tags[]`, `body[]` (array of paragraphs).
  Compute `minutes` honestly (~250 wpm).
- **Other sections** — skills, resume, navigation links — follow the same
  data-driven pattern: edit the source array, not a bespoke page.

Screenshots (if any) live in `public/screenshots/<id>/`. They are optional;
cards render fine without them.

---

## 2. Privacy scrub — BLOCKING; run before every deploy

This section is the heart of the skill. A public site under your real name is
a permanent record. Run every check below before staging a commit.

### 2a. No secrets, internal infrastructure, or PII in source

Run the grep below over your `src/` directory (adjust the path as needed).
It should print **CLEAN** or exit non-zero only on false positives you have
explicitly reviewed.

```bash
# Adapt <your-src-dir> to wherever your source files live.
rg -n --pcre2 \
  '(\b\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}\b|localhost|127\.0\.0\.1|\:\d{4,5}|password|passwd|api[_-]?key|Bearer |[A-Za-z0-9]{32,}|\d{3}[-.)\s]\s?\d{3}[-.\s]\d{4})' \
  <your-src-dir>/ || echo "CLEAN"
```

**Never ship any of the following:**

| Category | Examples |
|---|---|
| Internal IPs | `192.168.x.x`, `10.x.x.x`, `172.16–31.x.x` |
| Loopback / dev addresses | `localhost`, `127.0.0.1`, `0.0.0.0` |
| Internal ports | `:3000`, `:8080`, `:8123` (any `:<port>` reference) |
| Private hostnames | self-hosted git instances, tailnet hostnames, VPS IPs |
| Credentials | passwords, API keys, tokens, bearer strings |
| Long opaque strings | 32+ char alphanumeric strings that might be secrets |
| Phone numbers | any format |
| Home address / DOB / SSN | any of these |

**Fine to ship:** Tool and product names (e.g. Vercel, Cloudflare, Ollama,
Supabase). Coarse location (city, country). Stack names (Next.js, Python,
Flutter). What a system does and its scale — never its addresses.

### 2b. Email stays masked (anti-scrape)

Never hardcode a literal email address in source or templates. Email scrapers
harvest `user@domain.tld` patterns from static HTML and bundle files.

**Masking approach:**

1. In source, write the masked form using middots: `user·domain·tld`
   (middot `·` U+00B7, no `@`, no email pattern).
2. Ship a small client-side component (`MailGuard` or equivalent) that
   reassembles the real `mailto:` from a character-code array at runtime and
   patches any `href` or display text that contains the mask. Example pattern
   for the assembly (TypeScript):

   ```ts
   // Never write the address as a string literal — build it from char codes.
   // The minifier will NOT fold a .map(String.fromCharCode).join('') call,
   // so the literal never appears in the bundle.
   const chars = [117, 115, 101, 114, 64, 100, 111, 109, 97, 105, 110, 46, 116, 108, 100]; // fill in yours
   const address = chars.map(String.fromCharCode).join('');
   ```

3. Mount the component at layout/root level with a `MutationObserver` so it
   repairs links added by client-side routing too.

**Important:** if you ever inline `const addr = 'user' + '@' + 'domain.tld'`,
bundlers will constant-fold that into the literal. Use `.map(String.fromCharCode)`
instead.

### 2c. Blur third-party personal data in screenshots

Any screenshot showing real users' handles, display names, financial figures,
or authored content (e.g. from a community, forum, roster, or admin panel)
must be blurred before it goes into `public/screenshots/`.

Recommended approach: region-blur the content area with PIL/Pillow, keeping
the chrome and layout sharp so the UI value survives.

```python
# scripts/blur-screens.py (adapt paths and crop box to your screenshot)
from PIL import Image, ImageFilter

def blur_region(path_in, path_out, box):
    """box = (left, top, right, bottom) in pixels"""
    img = Image.open(path_in).convert("RGB")
    region = img.crop(box).filter(ImageFilter.GaussianBlur(radius=8))
    img.paste(region, box)
    img.save(path_out, quality=92)
```

After blurring, view the result and confirm:
- Personal data is illegible at 1x.
- The UI chrome / layout / feature showcase is still clear.

---

## 3. Verify — all checks must pass before commit

```bash
cd <your-site-repo>

# 1. TypeScript (if applicable) — must exit 0
npx tsc --noEmit

# 2. Production build — must exit 0
npm run build          # or: pnpm build / yarn build / your build command

# 3. Email literal must NOT appear in the output bundle
#    Adjust the output dir (.next/server, dist, out, _site, etc.)
grep -roh "user@domain\.tld" <build-output-dir> | wc -l   # must be 0
```

If the email literal count is > 0: the address leaked into the bundle, most
likely via string concatenation that the minifier folded. Switch to the
char-code array approach described in 2b.

---

## 4. Deploy

```bash
cd <your-site-repo>

# Stage and commit (conventional commit format)
git add <specific files>       # prefer explicit over git add -A
git commit -m "<type>: <what changed>"

# Push to your remote
git push origin <branch>       # replace with your remote and branch

# Deploy (replace with your deploy command)
<your deploy command>
# Examples:
#   npx vercel --prod --yes
#   netlify deploy --prod
#   gh workflow run deploy.yml
#   rsync -avz dist/ user@host:/var/www/html/
```

**Confirm live** — check HTTP status and verify the email literal is absent
from the live HTML:

```bash
# Check HTTP status for each new or changed route
for u in / /about /blog /projects; do
  curl -s -o /dev/null -w "%{http_code} $u\n" "https://<your-domain>$u"
done

# Confirm no literal email in the live homepage
curl -s "https://<your-domain>/" | grep -oc "user@domain"   # must be 0
```

---

## Self-improvement

After using this skill, update it with any new patterns you encountered:
a new PII category your grep missed, a build output path that changed, a
new way email can leak into a bundle. The scrub is only as good as its
last update.
