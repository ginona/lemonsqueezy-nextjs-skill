# Lemon Squeezy + Next.js + Vercel — Debug Skill

A Claude skill for debugging Lemon Squeezy payment flows in Next.js apps deployed on Vercel.

## What it covers
- Lemon Squeezy `custom_data` keys are snake_cased in webhooks — always
- Session state lost across serverless function instances
- Post-payment redirect dropping the path due to domain redirects
- Payment check bypassed by local dev flags left on in production
- Environment variable mismatches between local and deployed infra

## Install

\`\`\`bash
npx skills add tuuser/lemonsqueezy-nextjs-skill
\`\`\`

## Stack
Next.js 14 · App Router · Lemon Squeezy · Vercel · Supabase