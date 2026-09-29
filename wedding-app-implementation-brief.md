# Wedding Day App — Implementation Brief

**For:** Jesson & Diorra's wedding dinner — October 3, 2026, Park Hyatt Jakarta
**Stack:** Supabase (Postgres + Storage + Realtime) + Netlify (static hosting)
**Audience:** Dinner guests only — closed, mobile-first, no login required

## 1. What we're building

A single-page mobile web app with three tabs:
1. **Photo Hunt** — guests pick a photo category and upload a picture into it; completing the most categories wins a prize.
2. **Predictions** — guests submit guesses on a handful of fun questions about the night.
3. **Live Results** — a live bar-chart dashboard showing which answers are currently winning.

## 2. Assumptions made in this brief (confirm or change before building)

- **Photos go to Supabase Storage, not Google Drive.** This keeps everything in one stack and makes checklist completion directly queryable instead of manually checking a Drive folder. Easy to swap back to Drive links in section 5 if preferred.
- **No guest login/auth.** Name is a free-text field submitted with every action. Intentionally low-friction for a closed, low-stakes audience — duplicate or fake submissions are an acceptable risk, not worth building auth to prevent.
- **Simplest possible frontend.** One static HTML/CSS/JS page, no build step, using the Supabase JS client via CDN. A coding agent can scaffold it as a proper project (Vite, etc.) if that's preferred structurally — functionally it doesn't need to be more than a single page.

## 3. Data model (Supabase / Postgres)

Run in the Supabase SQL editor:

```sql
-- Predictions / votes
create table predictions (
  id uuid primary key default gen_random_uuid(),
  created_at timestamptz default now(),
  guest_name text not null,
  question_id text not null,
  answer text not null
);

-- Photo checklist claims (logged when a guest picks a category)
create table photo_claims (
  id uuid primary key default gen_random_uuid(),
  created_at timestamptz default now(),
  guest_name text not null,
  category text not null
);

alter table predictions enable row level security;
alter table photo_claims enable row level security;

create policy "anon can insert predictions" on predictions
  for insert to anon with check (true);
create policy "anon can read predictions" on predictions
  for select to anon using (true);

create policy "anon can insert photo_claims" on photo_claims
  for insert to anon with check (true);
create policy "anon can read photo_claims" on photo_claims
  for select to anon using (true);
```

**Storage bucket:**
- Create a bucket named `wedding-photos`, set to public.
- Upload path convention: `wedding-photos/{category-slug}/{timestamp}-{filename}`.
- Storage policies: allow `anon` `insert` (upload) and `select` (read) on this bucket. No `update`/`delete` for anon.

## 4. Realtime

Enable Postgres replication on the `predictions` table (Database → Replication in the Supabase dashboard) so the Live Results tab can subscribe to changes and update instantly instead of polling. If the coding agent wants to skip the realtime plumbing for a first pass, polling every 10–15 seconds is an acceptable fallback.

## 5. Feature specs

### Photo Hunt
- Category tiles: Bride & Groom Kissing, Mouthful of Food, Kids Running Around, Paparazzi Shot (someone photographing the couple), General Photos, plus a "more coming soon" placeholder — extensible list.
- Tapping a tile: collect the guest's name once (keep it in memory/sessionStorage for the rest of their visit), open a native `<input type="file" accept="image/*" capture>`, upload the file straight to `wedding-photos/{category}/...` in Supabase Storage, and insert a row into `photo_claims`.
- Prize judging is manual: after the event, the couple reviews `photo_claims` (and/or the storage bucket) to see who covered the most categories. No automated verification of photo *content* is expected or needed.

### Predictions
- Question set:
  - Who will cry first? — choice: Bride's Mom / Bride's Dad / Groom's Mom / Groom's Dad
  - How long will the best friend's speech be? — choice: 2 min / 3 min / 4 min
  - How many courses will dinner have? — number
  - How many pieces of fruit on the dessert table? — number
- One name field; submitting inserts one row per answered question into `predictions`.
- No resubmission guard needed.

### Live Results
- Read (or subscribe to) `predictions`, group by `question_id` + `answer`, render as horizontal bar charts (plain CSS width bars are enough — no charting library required) showing counts/percentages per answer.
- For the two numeric questions, group by exact guessed value; the closest guess to the real number wins, determined manually after the event.

## 6. Design direction

- Palette: deep ink/charcoal background (`#181310` / `#100c0a`), ivory content cards (`#f6efe2`), brass-gold accent (`#b8935a`).
- Typography: Playfair Display (italic, for the couple's names/headings) paired with Source Sans 3 for body/UI text.
- Mobile-first single page, three tabs: Photo Hunt / Predictions / Live Results.
- Avoid generic "AI wedding site" defaults — no all-caps eyebrow labels, no identical rounded SaaS cards everywhere; photo tiles and prediction cards should read as visually distinct from each other.

## 7. Deployment (Netlify)

- Static site — Supabase handles the entire backend, so there's no server to run.
- Deploy by drag-and-dropping the built folder into Netlify's dashboard, or connect a GitHub repo for continuous deployment.
- No secret environment variables needed beyond the Supabase project URL and **anon/public** key — these are safe to embed client-side by design. The RLS policies above are what actually control access, not key secrecy.
- Custom domain optional; Netlify's auto-generated `*.netlify.app` URL is enough for a QR code if time is short.

## 8. What the couple needs to do first

1. Create a free Supabase project and run the SQL from section 3 in its SQL editor; create the `wedding-photos` bucket and its policies.
2. Copy the project URL and anon public key from Supabase's API settings.
3. Hand this brief plus those two values to the coding agent to build and deploy to Netlify.

## 9. Out of scope / stretch ideas (not required for launch)

- Automated photo-content verification (e.g. detecting an actual kiss) — not realistic; keep prize judging manual.
- A public gallery of all uploaded photos — easy to add later since the bucket is already public-read, but not part of the initial build.
- Guest login/auth — deliberately skipped for a closed, one-night event.
