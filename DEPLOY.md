# Deploying Map Chaser to Vercel

1. Put THESE files at the root of a GitHub repo (package.json must be at the top level,
   not inside a folder), then import the repo in Vercel. Framework preset: Next.js.
2. Before the first deploy, add Environment Variables (Settings > Environment Variables):

   Required (the app will not build/run without these two):
   - NEXT_PUBLIC_SUPABASE_URL
   - NEXT_PUBLIC_SUPABASE_ANON_KEY

   Secret, server-only. Enter them in the Vercel dashboard yourself; never paste them into chat:
   - SUPABASE_SERVICE_ROLE_KEY  (Supabase > Project Settings > API). Needed for moderation,
     notifications, account deletion and showing user emails in admin.
   - ANTHROPIC_API_KEY          (console.anthropic.com). Without it every upload goes to manual review.
   - RESEND_API_KEY             (resend.com). Without it no emails are sent (actions still work).
   - NOTIFICATIONS_FROM_EMAIL   (optional, a sender on a domain verified in Resend)

3. NEXT_PUBLIC_* values are baked in at build time: after changing them, Redeploy.
4. In Supabase > Authentication > URL Configuration, set Site URL and Redirect URLs to your
   live domain. For Google sign-in, enable the provider and add your Client ID/Secret.
5. Become admin (SQL Editor), then visit /admin and set up 2FA:
     update public.profiles set role = 'admin'
     where id = (select id from auth.users where email = 'YOUR_EMAIL');
6. Existing accounts (created before the age gate) must confirm their date of birth at
   /complete-profile before they can upload or redeem.

NOTE: Supabase FREE projects are paused after about a week of inactivity, which takes the
whole site down until restored. For anything real, use a paid plan.
