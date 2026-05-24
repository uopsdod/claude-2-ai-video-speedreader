---
name: gmail-sign-in-setup
description: "End-to-end playbook for adding Google (Gmail) OAuth sign-in to a Next.js + Supabase project. Covers GCP project creation, OAuth consent screen, OAuth client credentials, Supabase Google provider config, code changes (login button + auth callback), identity linking, and common errors. Use when setting up Google sign-in on a new or existing Next.js + Supabase site."
---

# Google (Gmail) Sign-In Setup — Next.js + Supabase

Complete playbook for adding "Sign in with Google" to any Next.js App Router + Supabase project.

## Prerequisites

- A Next.js App Router project with Supabase auth already working (email/password sign-in)
- A Supabase project with `@supabase/ssr` configured (both server and browser clients)
- A Google account with access to GCP Console
- `gcloud` CLI installed (`brew install google-cloud-sdk`)

## Overview

The flow works like this:

1. User clicks "Sign in with Google" → calls `supabase.auth.signInWithOAuth({ provider: 'google' })`
2. Browser redirects to Google's OAuth consent screen
3. User grants permission → Google redirects to `https://<supabase-ref>.supabase.co/auth/v1/callback`
4. Supabase exchanges the code, creates/links the user, redirects to your app's `redirectTo` URL
5. Your app's auth callback route calls `exchangeCodeForSession(code)` to set the session cookie

## Step 1: GCP — Create Project & OAuth Credentials

### 1a. Create a GCP project

```bash
source "$(brew --prefix)/share/google-cloud-sdk/path.zsh.inc"
gcloud auth login
gcloud projects create <project-id> --name="<project-name>"
gcloud config set project <project-id>
```

### 1b. Configure OAuth consent screen

Go to: `https://console.cloud.google.com/auth/consent?project=<project-id>`

- User Type: **External**
- App name: your site name
- User support email: your email
- Authorized domain: your domain (e.g. `example.com`)
- Developer contact email: your email
- Scopes: `email` and `profile` (defaults)

Note: `email` and `profile` are basic scopes, NOT sensitive/restricted. The "100 user cap" shown in the console does NOT apply to basic scopes.

### 1c. Create OAuth Client ID

Go to: `https://console.cloud.google.com/apis/credentials/oauthclient?project=<project-id>`

- Application type: **Web application**
- Name: descriptive name (e.g. `MySite Supabase`)
- Authorized redirect URI: `https://<supabase-project-ref>.supabase.co/auth/v1/callback`

Find your Supabase project ref in your `.env.local`:
```bash
grep NEXT_PUBLIC_SUPABASE_URL .env.local
# Output: NEXT_PUBLIC_SUPABASE_URL=https://<ref>.supabase.co
```

Save the **Client ID** and **Client Secret**.

### 1d. Publish the app

Still on the OAuth consent screen page, click **"Publish App"** to move from Testing → Production. Without this, only manually-added test users (up to 100) can sign in.

## Step 2: Supabase — Enable Google Provider

Go to: `https://supabase.com/dashboard/project/<project-ref>/auth/providers`

1. Find **Google** in the provider list and **enable** it
2. Enter the **Client ID** and **Client Secret** from Step 1c
3. Check **"Skip nonce checks"** (required for server-side PKCE flow used by `@supabase/ssr`)
4. Save

## Step 3: Code Changes

### 3a. Add Google sign-in button to login page

The button must use the **browser** Supabase client (not server) since `signInWithOAuth` triggers a browser redirect.

```tsx
'use client'

import { createClient } from '@/lib/supabase-browser'  // your browser client

function GoogleSignInButton({ next }: { next: string }) {
  return (
    <button
      type="button"
      onClick={async () => {
        const supabase = createClient()
        await supabase.auth.signInWithOAuth({
          provider: 'google',
          options: {
            redirectTo: `${window.location.origin}/auth/confirm?next=${encodeURIComponent(next)}`,
          },
        })
      }}
    >
      <svg width="18" height="18" viewBox="0 0 24 24" aria-hidden="true">
        <path d="M22.56 12.25c0-.78-.07-1.53-.2-2.25H12v4.26h5.92a5.06 5.06 0 0 1-2.2 3.32v2.77h3.57c2.08-1.92 3.28-4.74 3.28-8.1z" fill="#4285F4" />
        <path d="M12 23c2.97 0 5.46-.98 7.28-2.66l-3.57-2.77c-.98.66-2.23 1.06-3.71 1.06-2.86 0-5.29-1.93-6.16-4.53H2.18v2.84C3.99 20.53 7.7 23 12 23z" fill="#34A853" />
        <path d="M5.84 14.09c-.22-.66-.35-1.36-.35-2.09s.13-1.43.35-2.09V7.07H2.18A11.96 11.96 0 0 0 0 12c0 1.94.46 3.77 1.28 5.4l3.56-2.77.01-.54z" fill="#FBBC05" />
        <path d="M12 5.38c1.62 0 3.06.56 4.21 1.64l3.15-3.15C17.45 2.09 14.97 1 12 1 7.7 1 3.99 3.47 2.18 7.07l3.66 2.84c.87-2.6 3.3-4.53 6.16-4.53z" fill="#EA4335" />
      </svg>
      Sign in with Google
    </button>
  )
}
```

Key: `redirectTo` uses `window.location.origin` so it works on both localhost and production without configuration.

### 3b. Auth callback route — handle OAuth code exchange

File: `app/auth/confirm/route.ts` (or wherever your auth callback lives)

**Critical bug to avoid:** Do NOT call `supabase.auth.signOut()` before `exchangeCodeForSession()`. The sign-out clears the PKCE code verifier cookie that Supabase stores client-side, causing the code exchange to fail silently.

Correct pattern:

```tsx
import { type NextRequest, NextResponse } from 'next/server'
import { createClient } from '@/lib/supabase-server'

export async function GET(request: NextRequest) {
  const { searchParams } = new URL(request.url)
  const code = searchParams.get('code')
  const token_hash = searchParams.get('token_hash')
  const type = searchParams.get('type') as 'signup' | 'recovery' | 'email'
  const next = searchParams.get('next') ?? '/dashboard'

  // Validate next param
  if (!next.startsWith('/') || next.startsWith('//') ||
      next.startsWith('/api') || next.startsWith('/auth')) {
    return NextResponse.redirect(new URL('/login?error=Invalid+redirect', request.url))
  }

  const supabase = await createClient()

  // OAuth flow — do NOT sign out first (breaks PKCE)
  if (code) {
    const { error } = await supabase.auth.exchangeCodeForSession(code)
    if (!error) return NextResponse.redirect(new URL(next, request.url))
    console.error('[auth/confirm] exchangeCodeForSession failed:', error.message)
    return NextResponse.redirect(
      new URL('/login?error=' + encodeURIComponent(error.message), request.url),
    )
  }

  // Email confirmation flow — safe to sign out first here
  if (token_hash && type) {
    await supabase.auth.signOut()
    const { error } = await supabase.auth.verifyOtp({ token_hash, type })
    if (!error) return NextResponse.redirect(new URL(next, request.url))
  }

  return NextResponse.redirect(new URL('/login?error=Could+not+verify', request.url))
}
```

## Step 4: Identity Linking

When a user signs up with email/password and later signs in with Google (same email), Supabase can either:

- **Create two separate users** (bad — different IDs, different data)
- **Link identities to one user** (good — same ID, both sign-in methods work)

Check your setting at: `https://supabase.com/dashboard/project/<ref>/settings/auth` → **"User Identity Linking"**

Enable **"Automatic Linking"** so users with the same email share one account regardless of sign-in method.

Verify it works by checking `app_metadata.providers` on a user:

```bash
curl -s "https://<ref>.supabase.co/auth/v1/admin/users?page=1&per_page=50" \
  -H "apikey: $SUPABASE_SECRET_KEY" \
  -H "Authorization: Bearer $SUPABASE_SECRET_KEY" | python3 -c "
import sys, json
for u in json.load(sys.stdin).get('users', []):
    if u.get('email') == 'test@example.com':
        print(json.dumps(u['app_metadata'], indent=2))
        break
"
```

Expected output for a linked user:
```json
{
  "provider": "email",
  "providers": ["email", "google"]
}
```

## Step 5: Deploy & Test

1. Deploy to production
2. Test sign-in at `https://yoursite.com/login`
3. Verify the user appears in Supabase Auth dashboard with both providers linked

No extra Vercel/hosting env vars needed — the OAuth flow goes through Supabase's callback URL (`supabase.co/auth/v1/callback`), not your app's origin.

## Common Errors

| Error | Cause | Fix |
|-------|-------|-----|
| `redirect_uri_mismatch` (Error 400) | Redirect URI in GCP doesn't exactly match | Verify URI is exactly `https://<ref>.supabase.co/auth/v1/callback` — no trailing slash, no typo |
| `Could not verify email` | `signOut()` called before `exchangeCodeForSession()` | Remove signOut from the OAuth code branch (see Step 3b) |
| `Access blocked: This app's request is invalid` | OAuth consent screen still in Testing mode | Publish the app in GCP console (Step 1d) |
| Two accounts for same email | Automatic identity linking is off | Enable in Supabase: Settings → Auth → User Identity Linking → Automatic |
| `insufficient authentication scopes` | Trying to use GCP Site Verification API | This API requires a special OAuth scope; use Google Search Console UI instead |

## Optional: Custom Domain

Supabase Pro ($25/mo) + Custom Domain add-on ($10/mo) replaces `<ref>.supabase.co` with `auth.yourdomain.com` in the OAuth redirect. Purely cosmetic — users see it flash briefly during the redirect. Not required for functionality. Skip until revenue justifies it.

## Optional: Branded OAuth Consent Screen

To show your domain name on the Google consent screen:

1. Verify domain ownership via Google Search Console (`https://search.google.com/search-console/welcome`)
2. Add the DNS TXT record to your DNS provider
3. Add verified domain as an authorized domain in the OAuth consent screen settings
