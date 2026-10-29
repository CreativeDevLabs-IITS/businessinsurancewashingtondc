# Value Insurance Agency Inc. Website

Website for Value Insurance Agency Inc. (businessinsurancewashingtondc.com), an independent insurance agency in Washington, DC. It is a [Next.js](https://nextjs.org) app (React, Tailwind CSS, GSAP and Lenis smooth scrolling) that is **exported as a static site**: `npm run build` writes plain HTML, CSS and JavaScript into a `dist/` folder, so no Node.js server is needed to run it.

## Quick start

```bash
npm ci               # install dependencies (first time, or after package.json changes)
npm run build        # creates ./dist
```

**Point your domain's web server root at the `dist/` folder**, for example `/var/www/businessinsurancewashingtondc/dist`. The full Lightsail steps are below.

## Project structure

```
src/app/             Pages (home, coverage/[slug], glossary, about, refer, newsletter, privacy), layout, styles
src/components/      Sections and UI components
src/data/            content.ts (site text and coverage pages) and glossary.ts
public/              Images and logo, copied into dist/
scripts/             redirects.mjs (runs after every build) and the content/image helper scripts
next.config.ts       Static export settings (output: "export", distDir: "dist")
deploy/              fix-nginx.sh, see the Nginx section below
dist/                Build output. Point your domain here. Not committed to git.
REDIRECTS.md         List of old addresses and where they now go (regenerated on every build)
```

## Old site addresses

The previous site used `.aspx` addresses such as `/default.aspx`, `/business/default.aspx` and `/category.aspx?id=COMMERCIAL`. Every build (`postbuild` runs `scripts/redirects.mjs`) writes a small redirect page for each of them into `dist/`, so old links and search results keep working. `deploy/fix-nginx.sh` (step 3 below) makes Nginx serve those pages. The list is in `REDIRECTS.md`.

## Run locally

Requires Node.js 20.9 or newer.

```bash
npm ci
npm run dev       # development server at http://localhost:3000
npm run build     # creates ./dist
npm start         # serves ./dist at http://localhost:3000
```

## Deploy to AWS Lightsail

These steps start **after you have cloned this repo on the Lightsail server** (as the `ubuntu` user, into `/var/www/businessinsurancewashingtondc`). Node.js and Nginx are set up for you by `sudo cpanel sites new`, so there is nothing to install by hand.

### 1. Build the site

```bash
cd /var/www/businessinsurancewashingtondc
npm ci
npm run build
```

The build downloads the site's Google fonts, so the server needs internet access while it runs. If this fails with `EACCES: permission denied`, see "If `npm run build` says permission denied" below.

This creates `/var/www/businessinsurancewashingtondc/dist`, the folder your domain will point at.

### 2. Create the site with `cpanel sites new`

```bash
sudo cpanel sites new
```

Answer the prompts like this (use your real domain):

| Prompt | Answer |
|--------|--------|
| Select application type | `4` (Static / React SPA) |
| Enter domain name | `example.com` |
| Enter site folder directory | `/var/www/businessinsurancewashingtondc/dist` |
| Apply secure ownership and permissions now? | `n` (so `ubuntu` can keep rebuilding). If you answer `y`, run `sudo chown -R ubuntu:ubuntu /var/www/businessinsurancewashingtondc` afterwards. |

The command writes the Nginx config for the domain (root set to the `dist` folder) and reloads Nginx.

### 3. Let Nginx serve the site's JavaScript and pages (required)

The `cpanel sites new` template only allows `.js` files under `/assets/` or `/static/`, returns 404 for other file types, and serves the home page for any URL that is not a file. Next.js keeps its JavaScript in `/_next/static/` and this site has several pages, so **without this step the page loads but has no animations, no menu and hidden sections, and inner pages show the home page**. Run once per domain:

```bash
cd /var/www/businessinsurancewashingtondc
sudo bash deploy/fix-nginx.sh example.com
```

See "If the site loads but animations or pages are missing" below for details.

### 4. Open the firewall in Lightsail

In the Lightsail console, open your instance, then **Networking**, then **IPv4 Firewall**. Make sure these rules exist: **HTTP (TCP 80)** and **HTTPS (TCP 443)**. Add the same rules under the IPv6 firewall if you use IPv6.

### 5. Point your domain at the server

1. In Lightsail, open **Networking**, then **Create static IP**, and attach it to the instance. Without this, the IP changes whenever the instance restarts.
2. At your domain registrar (or in the Lightsail DNS zone, or Cloudflare), create these records pointing to the static IP:

   | Type | Name | Value |
   |------|------|-------|
   | A | `@` | your static IP |
   | A | `www` | your static IP |

3. Wait for DNS to propagate (usually a few minutes, sometimes up to an hour). Check with `dig +short example.com`.

### 6. Check HTTPS

`cpanel sites new` sets up the HTTPS certificate for the domain. Once DNS points at the server, open `https://example.com`. If the browser shows no padlock or a certificate warning, get a certificate with Certbot:

```bash
sudo certbot --nginx -d example.com -d www.example.com
```

## Updating the site

After you push changes to GitHub, run this on the server:

```bash
cd /var/www/businessinsurancewashingtondc
git pull
npm ci
npm run build
```

Nginx serves the files straight from `dist`, so there is nothing to restart. Run these from the repo folder, not from inside `dist/`.

## If `npm run build` says "permission denied" (EACCES)

```
Error: EACCES: permission denied, mkdir '/var/www/businessinsurancewashingtondc/dist'
```

The build has to create a fresh `dist/` folder inside the repo folder, so the user running it (`ubuntu`) must own that folder. It does not when the repo was cloned with `sudo`, or when the `cpanel sites new` tool's "Apply secure ownership and permissions now?" prompt was answered `y`. Nginx does not need ownership, only read access, so giving the folder to `ubuntu` is safe:

```bash
ls -ld /var/www/businessinsurancewashingtondc /var/www/businessinsurancewashingtondc/dist      # shows who owns the folder (should say ubuntu)
sudo chown -R ubuntu:ubuntu /var/www/businessinsurancewashingtondc
cd /var/www/businessinsurancewashingtondc
npm ci
npm run build
```

The fix is per folder, so run it once for every site on the server. To fix several at once (change the names to match your folders; it is safe to run again on sites that are already fixed):

```bash
for s in businessinsurancewashingtondc mickey-academy dl-animal stjohn-vet animalhealth-center proactive-therapy bestfit-tailor kidderstreet-drycleaners deluxe-drycleaners littlebears-childcare; do
  [ -d /var/www/$s ] && sudo chown -R ubuntu:ubuntu /var/www/$s && echo "fixed $s"
done
```

If the error comes back later, something changed the ownership again (usually answering `y` to the `cpanel sites new` secure-ownership prompt, or running `git pull` or the build with `sudo`). Run the `chown` again.

Tips:

- Clone as `ubuntu` (no `sudo`). If `/var/www` is not writable, create the folder once with `sudo mkdir /var/www/businessinsurancewashingtondc && sudo chown ubuntu:ubuntu /var/www/businessinsurancewashingtondc` and then clone into it.
- Run `npm run build` from the repo folder, not from inside `dist/`. The build replaces `dist/`, which leaves a shell that is sitting inside it pointing at a deleted folder.

## If the site loads but animations or pages are missing (`cpanel sites new`)

Sites created with the server's `cpanel sites new` tool get an Nginx config that only allows `.js` files under `/assets/` or `/static/`, returns **404 for every other file extension**, and serves the home page for URLs such as `/privacy/`. Fix it once per site on the server from the repo folder (replace the domain with yours):

```bash
cd /var/www/businessinsurancewashingtondc
sudo bash deploy/fix-nginx.sh example.com
```

The script backs up the Nginx config (to `/root/example.com.nginx.bak`), adds rules that allow `.js`, `.mjs`, `.html`, `.txt`, `.xml`, `.json`, `.pdf`, `.xlsx`, `.docx` `.aspx` (for the old site addresses), and a few media files, makes inner pages resolve to their own `index.html`, tests the config, and reloads Nginx. If the test fails it puts the backup back. Running it twice is harmless.

Afterwards hard-refresh the site (Ctrl+Shift+R), and if the domain is behind Cloudflare, purge its cache (Caching, then Purge Everything). If you run `cpanel sites new` again for the same domain, it rewrites the config, so run the script again.

To check it worked, open the site with the browser's developer tools (F12) and look at the Console: there should be no red 404 errors for `/_next/static/...` files, and `/privacy/` should show the Privacy page.

## Troubleshooting

- **`npm run build` fails with `EACCES: permission denied`:** the repo folder is not owned by `ubuntu`. Run `sudo chown -R ubuntu:ubuntu /var/www/businessinsurancewashingtondc`.
- **Page loads but animations do not run, or `/_next/static/...` files return 404:** run `deploy/fix-nginx.sh` (step 3).
- **Inner pages (`/privacy/`, `/coverage/business-insurance/`) show the home page:** run `deploy/fix-nginx.sh` (step 3).
- **`npm run build` prints `Killed`:** the server ran out of memory. Add swap (for example `sudo fallocate -l 2G /swapfile && sudo chmod 600 /swapfile && sudo mkswap /swapfile && sudo swapon /swapfile`) and run the build again.
- **403 Forbidden:** Nginx cannot read the files. Run `sudo chmod -R o+rX /var/www/businessinsurancewashingtondc/dist`, and make sure the folders above it are readable.
- **Default Nginx welcome page shows:** the domain has no site yet. Run `sudo cpanel sites new` for it (step 2).
- **Site not loading on the domain:** check the DNS records, the Lightsail firewall ports (80 and 443), and `sudo nginx -t`.
- **Changes not showing:** run `npm run build` again, hard-refresh the browser (Ctrl+Shift+R) and purge the Cloudflare cache. Static files are cached for 30 days.
- **Logs:** `sudo tail -f /var/log/nginx/error.log`

## Notes

- **Static export:** the site is exported with `output: "export"`, `distDir: "dist"` and `trailingSlash: true` in `next.config.ts`, and images use `unoptimized: true` because Next.js image optimization needs a server. Server features (API routes, server actions, middleware, `cookies()` and `headers()`) cannot be used with this setup.
- **Next.js version:** `next` is pinned to 16.4.3. Versions 16.0.0 to 16.4.0 have critical security advisories, and `next build` stops with an error on them.
- **Forms:** the contact, referral and newsletter forms have no backend. Submitting one opens the visitor's email app with a prefilled message to the agency email set in `src/data/content.ts`. To receive messages directly, connect a form service (for example Formspree) in `src/components/ui/ContactForm.tsx` and `SimpleForm.tsx`.
- **Fonts:** Fraunces, Inter Tight and JetBrains Mono are downloaded from Google Fonts at build time and bundled into `dist/`.
- **Helper scripts:** the `content:*` npm scripts (scraping, glossary, images, icons) were used to prepare the content and are not needed to deploy.
