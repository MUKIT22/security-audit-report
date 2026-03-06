# Website Security Audit Report
**Prepared by:** Mukit Muntasir 
**Date:** March 6, 2026  
**Scope:** Basic infrastructure security audit of 5 websites   

---

## Sites Audited

| # | Website | URL |
|---|---------|-----|
| 1 | Héctor Quiroga – Abogado de Inmigración | https://abogadoquiroga.com |
| 2 | Abogada Quiroga | https://abogadaquiroga.com |
| 3 | Quiroga Law Office, PLLC | https://www.quirogalawoffice.com |
| 4 | Hector's Whips | https://www.hectorswhips.com |
| 5 | US Digital Marketing Solutions | https://usdigitalmarketingsolutions.com |

---

## Methodology

Checks performed:
- HTTPS / SSL certificate validation (live request)
- HTTP → HTTPS redirect behavior
- Admin login page exposure (`/wp-admin/`)
- CMS / platform detection
- `robots.txt` analysis for information disclosure
- `xmlrpc.php` exposure (WordPress brute-force attack vector)
- `readme.html` exposure (WordPress version disclosure)
- Plugin file exposure via `robots.txt` references
- Contact form analysis for CAPTCHA/spam protection
- Public directory indicators
- External resource / CDN exposure

---

## Risk Level Definitions

| Level | Meaning |
|-------|---------|
| 🔴 HIGH | Can be exploited directly; immediate action required |
| 🟡 MEDIUM | Increases attack surface; fix within days |
| 🟢 LOW | Best-practice hardening; fix when possible |

---

# Site 1 — abogadoquiroga.com

**Platform:** Custom CMS (non-WordPress)  
**Hosting:** Cloud-based (media served via AWS S3 + CloudFront CDN)  

## Checks

| Check | Status | Notes |
|-------|--------|-------|
| HTTPS active | ✅ PASS | Site loads on HTTPS |
| HTTP → HTTPS redirect | ✅ PASS | Standard redirect active |
| SSL/TLS certificate | ✅ PASS | Valid certificate present |
| WordPress installed | ✅ N/A | This is a custom site |
| `robots.txt` hardened | ✅ PASS | Disallows `/admin/`, `/auth/`, `/analytics/`, `/media-library/`, JSON endpoints |
| CAPTCHA on contact forms | 🚨 FAIL | No reCAPTCHA, hCaptcha |
| Public S3 bucket exposure | ⚠️ FLAGGED | Media served from named bucket: `argosproduction.s3.us-west-2.amazonaws.com` |

## Findings

### Finding 1.1 — S3 Bucket Name Exposed in HTML Source
- **Risk:** 🟢 LOW  
- **Detail:** All media (images, thumbnails) loads from the publicly named S3 bucket `argosproduction.s3.us-west-2.amazonaws.com`. If this bucket were misconfigured to allow public listing or writes, it would be a significant issue.  
- **Recommended Fix:** Verify the S3 bucket policy in AWS IAM:
  - Confirm `s3:ListBucket` is **denied** for public/anonymous access.
  - Enable **Block Public Access** settings at the bucket level.
  - Consider routing all media through a CloudFront CDN URL with an Origin Access Control (OAC) policy to hide the direct S3 bucket name.

### Finding 1.2 — CAPTCHA on Contact Forms Unknown
- **Risk:** 🟡 MEDIUM  
- **Detail:** The site includes multiple contact/consultation forms (navigation links to `/contacto/`). Without CAPTCHA, these can be exploited for spam or lead harvesting.  
- **Recommended Fix:** Implement reCAPTCHA v3 (invisible) or hCaptcha on all forms collecting user data.

### Finding 1.3 — CDN Dependency (CloudFront)
- **Risk:** 🟢 LOW  
- **Detail:** Podcast thumbnails load from `d3t3ozftmdmh3i.cloudfront.net`. This is a shared CloudFront distribution. Ensure the origin is restricted and access logs are enabled.  
- **Recommended Fix:** Review CloudFront distribution settings; ensure geo-restrictions and logging are enabled.

---

# Site 2 — abogadaquiroga.com

**Platform:** WordPress  
**Hosting:** Managed WordPress hosting (HTTPS served)  

## Checks

| Check | Status | Notes |
|-------|--------|-------|
| HTTPS active | ✅ PASS | Site loads on HTTPS |
| HTTP → HTTPS redirect | ✅ PASS | Active |
| SSL/TLS certificate | ✅ PASS | Valid |
| `wp-admin` exposed | 🚨 FAIL | Login page fully accessible at `/wp-admin/` |
| `wp-login.php` exposed | 🚨 FAIL | Accessible via standard WP login page |
| WordPress version visible | ✅ PASS  | Wordpress version is not exposed publically
| `xmlrpc.php` status | ⚠️ FLAGGED | Returns non-standard response (may be partially reachable) |
| Plugin file exposed | 🚨 FAIL | `robots.txt` reveals `/wp-content/uploads/wpo/wpo-plugins-tables-list.json` — WP-Optimize plugin confirmed |
| `robots.txt` wp-admin rule | ✅ PASS | `Disallow: /wp-admin/` for bots |
| Login attempt limitation | ✅ PASS | Block multiple login attempts

## Findings

### Finding 2.1 — WordPress Admin Login Page Publicly Accessible
- **Risk:** 🔴 HIGH  
- **Detail:** Navigating to `https://abogadaquiroga.com/wp-admin/` returns a fully functional WordPress login form. This exposes the site to brute-force and credential-stuffing attacks.  
- **Recommended Fix:**
  - Install a security plugin such as **Wordfence** or **Solid Security (iThemes Security)**.
  - Rename or relocate the login URL using a plugin like **WPS Hide Login**.
  - Add IP allowlisting to restrict `/wp-admin/` access to known IP addresses via `.htaccess`:
    ```apache
    <Limit GET POST>
      Order deny,allow
      Deny from all
      Allow from YOUR.OFFICE.IP.ADDRESS
    </Limit>
    ```
  - Implement **Two-Factor Authentication (2FA)** for all admin accounts.

### Finding 2.2 — Plugin Identity Disclosed in robots.txt
- **Risk:** 🟡 MEDIUM  
- **Detail:** The `robots.txt` file contains the line:
  ```
  Disallow: /wp-content/uploads/wpo/wpo-plugins-tables-list.json
  ```
  This confirms the **WP-Optimize** plugin is installed and reveals its version data file path. Attackers can use this to identify and target the specific plugin version.  
- **Recommended Fix:**
  - Remove the disallow rule from `robots.txt` (the better approach is to prevent the file from being accessible, not just hiding it from bots).
  - Ensure WP-Optimize is **up to date**.
  - Add a server rule to deny the JSON file from being served publicly:
    ```apache
    <Files "wpo-plugins-tables-list.json">
      Require all denied
    </Files>
    ```

### Finding 2.3 — xmlrpc.php May Be Partially Accessible
- **Risk:** 🟡 MEDIUM  
- **Detail:** `xmlrpc.php` returned a non-content response (not a 404). This file is a legacy WordPress endpoint used for remote publishing; it is a well-known brute-force amplification target (one request can perform thousands of login attempts via `system.multicall`).  
- **Recommended Fix:** Disable XML-RPC via `.htaccess`:
  ```apache
  <Files xmlrpc.php>
    Require all denied
  </Files>
  ```
  Or block it via a security plugin (Wordfence has a one-click option).


---

# Site 3 — quirogalawoffice.com

**Platform:** WordPress  
**Hosting:** Managed WordPress hosting  

## Checks

| Check | Status | Notes |
|-------|--------|-------|
| HTTPS active | ✅ PASS | Site loads on HTTPS |
| HTTP → HTTPS redirect | ✅ PASS | Active |
| SSL/TLS certificate | ✅ PASS | Valid |
| `wp-admin` exposed | 🚨 FAIL | Login page fully accessible at `/wp-admin/` |
| `wp-login.php` exposed | 🚨 FAIL | Standard WP login accessible |
| WordPress version visible | 🚨 FAIL  | WordPress 6.8.3 not the latest version |
| `xmlrpc.php` status | ⚠️ FLAGGED | Returns non-standard response (similar to Site 2) |
| `robots.txt` wp-admin rule | ✅ PASS | `Disallow: /wp-admin/` for bots |
| Login attempt limitation | ✅ PASS | Blocks multiple login attempts|
| CAPTCHA on forms | ✅ PASS  | CAPTCHA seems present|

## Findings

### Finding 3.1 — WordPress Admin Login Page Publicly Accessible
- **Risk:** 🔴 HIGH  
- **Detail:** `https://www.quirogalawoffice.com/wp-admin/` returns the full WordPress login page. This site handles sensitive legal client data, making an exposed admin panel especially high-risk.  
- **Recommended Fix:** Same as Site 2 — implement login page relocation, IP restriction, 2FA, and a security plugin immediately. Priority is higher here given the sensitivity of client data (immigration case files, contact data).

### Finding 3.2 — xmlrpc.php May Be Accessible
- **Risk:** 🟡 MEDIUM  
- **Detail:** The `xmlrpc.php` file returned a non-empty response. If reachable, this enables brute-force attacks and potential SSRF.  
- **Recommended Fix:** Same as Site 2 — block via `.htaccess` or security plugin.

### Finding 3.4 — WordPress Version Visible
- **Risk:** 🔴 HIGH  
- **Detail:** The site is running WordPress 6.8.3, which is publicly visible via the meta generator tag or other site endpoints. This version is not the latest release, exposing known vulnerabilities that attackers could exploit.  
- **Recommended Fix:** Update WordPress core to the latest stable version immediately.



---

# Site 4 — hectorswhips.com

**Platform:** Custom Web Application (non-WordPress)  
**Hosting:** Cloud-hosted (CDN: `cdn.hectorswhips.com`)  

## Checks

| Check | Status | Notes |
|-------|--------|-------|
| HTTPS active | ✅ PASS | Site loads on HTTPS |
| HTTP → HTTPS redirect | ✅ PASS | Active |
| SSL/TLS certificate | ✅ PASS | Valid |
| `wp-admin` exposed | ✅ PASS | Returns custom 404 — no WordPress |
| Admin portal | ⚠️ FLAGGED | Admin/user auth portal at `/portal/auth` |
| Login attempt limitation | 🚨 FAIL | Does not Blocks multiple login attempts|
| WordPress installed | ✅ N/A | Custom platform |
| `robots.txt` | ⚠️ FLAGGED | Only `Allow: /` and sitemap — no admin path protections |
| CAPTCHA on forms | ❓ UNKNOWN | Booking form present; CAPTCHA not confirmed |


## Findings

### Finding 4.1 — Custom Auth Portal Exposed at /portal/auth
- **Risk:** 🟡 MEDIUM  
- **Detail:** The user/admin authentication portal is accessible at `https://www.hectorswhips.com/portal/auth`. This is not hidden or protected from bots, and the path is published in the site's navigation.  
- **Recommended Fix:**
  - Implement rate-limiting on the `/portal/auth` login endpoint (maximum 5 attempts before lockout).
  - Add CAPTCHA (hCaptcha or Cloudflare Turnstile) to the login form.
  - Implement 2FA for all admin/owner accounts.
  - Consider adding a `Disallow: /portal/` rule in `robots.txt`.

### Finding 4.2 — robots.txt Has No Access Restrictions
- **Risk:** 🟢 LOW  
- **Detail:** The `robots.txt` file only contains `Allow: /` — no sensitive paths are restricted. While this doesn't create a vulnerability by itself, it is a best-practice gap.  
- **Recommended Fix:** Add `Disallow: /portal/` and any other administrative or private paths to `robots.txt`.



### Finding 4.3 — No Security Headers Confirmed
- **Risk:** 🟡 MEDIUM  
- **Detail:** Without raw HTTP response access, security headers (CSP, HSTS, X-Frame-Options, X-Content-Type-Options, Referrer-Policy) cannot be confirmed for this custom platform.  
- **Recommended Fix:** Run the site through **https://securityheaders.com** and implement any missing headers. Recommended minimum:
  - `Strict-Transport-Security: max-age=31536000; includeSubDomains; preload`
  - `X-Frame-Options: DENY`
  - `X-Content-Type-Options: nosniff`
  - `Referrer-Policy: strict-origin-when-cross-origin`
  - `Content-Security-Policy` (tailored to the app)

---

# Site 5 — usdigitalmarketingsolutions.com

**Platform:** WordPress  
**Hosting:** Managed WordPress hosting  

## Checks

| Check | Status | Notes |
|-------|--------|-------|
| HTTPS active | ✅ PASS | Site loads on HTTPS |
| HTTP → HTTPS redirect | ✅ PASS | Active |
| SSL/TLS certificate | ✅ PASS | Valid |
| `wp-admin` exposed | ✅ PASS | `/wp-admin/` redirects to homepage (appears protected) |
| WordPress installed | ✅ CONFIRMED | latest WordPress 6.9.1 |
| `robots.txt` configuration | ⚠️ FLAGGED | Blocks ALL web crawlers with `User-agent: * Disallow: /` |
| Login attempt limitation | ✅ PASS  |


## Findings

### Finding 5.1 — robots.txt Blocks All Crawlers (SEO Risk + Unusual Configuration)
- **Risk:** 🟡 MEDIUM  
- **Detail:** The `robots.txt` contains `User-agent: * Disallow: /` which blocks ALL web crawlers from indexing the site. While this is not a direct security vulnerability, it is an unusual and potentially unintentional configuration for a marketing agency. It may also indicate the config was copy-pasted without verification.  
  ```
  User-agent: *
  Disallow: /
  ```
  This also makes it harder to detect public exposure of files since standard scanners are blocked.  
- **Recommended Fix:** Review and correct the `robots.txt` file. If the intent is to allow search engine indexing (expected for a marketing agency website), change to:
  ```
  User-agent: *
  Disallow: /wp-admin/
  Allow: /wp-admin/admin-ajax.php
  ```


---





