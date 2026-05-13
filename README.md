# QuickHire (local development)

QuickHire is a Vite + React job portal. These steps get the app running locally.

1. Copy environment template and fill values:

```bash
cp .env.example .env
# then edit .env and replace placeholders
```

2. Install dependencies and run dev server:

```bash
npm install
npm run dev
```

3. Open the app: http://localhost:5173/

Notes:
- The app expects `VITE_CLERK_PUBLISHABLE_KEY`, `VITE_SUPABASE_URL`, and `VITE_SUPABASE_ANON_KEY` in `.env`.
- If you see an error about missing publishable key, ensure `VITE_CLERK_PUBLISHABLE_KEY` is set.
