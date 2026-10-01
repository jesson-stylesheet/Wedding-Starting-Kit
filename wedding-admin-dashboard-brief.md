# Prediction Market & Event Admin Dashboard — Implementation Brief

**For:** Jesson & Diorra's Wedding Celebration — October 3, 2026, Park Hyatt Jakarta  
**Purpose:** Private host/couple control panel (`/admin.html`) to:
1. Settle all 10 multiple-choice prediction questions by marking the correct answers.
2. Automatically compute guest scores and display the **Live Winner Leaderboard** (ready for the MC / prize announcement).
3. Review **Photo Hunt completion counts** per guest to declare the photo prize winner.
4. Provide emergency control (lock/unlock predictions submissions, clear demo data, export CSV).

**Stack:** Supabase (Postgres + Realtime) + Netlify Static Hosting (same project as `index.html`).

---

## 1. Access & Security

- **Route:** Private unlinked file `/admin.html` (not linked in the main guest navigation).
- **Access Guard (Client-side PIN):** Simple 4-digit PIN gate (e.g. `1003` for the wedding date Oct 3) stored in memory/sessionStorage so random curious guests who guess the URL cannot tamper with answers from their phones.
- **Audience:** Jesson, Diorra, or the designated Best Man / Bridesmaid / Wedding Organizer running the MC coordination.

---

## 2. Supabase Data Model Updates

Run this in the Supabase SQL Editor to support answer settlement and market control:

```sql
-- Store correct answers set by the couple/admin
create table if not exists question_answers (
  question_id text primary key,
  correct_answer text not null,
  set_at timestamptz default now()
);

-- Optional market lock flag (e.g., locking predictions when dinner starts)
create table if not exists market_settings (
  key text primary key,
  value text not null,
  updated_at timestamptz default now()
);

-- Default setting: predictions open
insert into market_settings (key, value)
values ('predictions_locked', 'false')
on conflict (key) do nothing;

-- Enable RLS
alter table question_answers enable row level security;
alter table market_settings enable row level security;

-- Policies for question_answers
create policy "anon can read correct answers" on question_answers
  for select to anon using (true);
create policy "anon can write correct answers" on question_answers
  for all to anon using (true) with check (true);

-- Policies for market_settings
create policy "anon can read market settings" on market_settings
  for select to anon using (true);
create policy "anon can write market settings" on market_settings
  for all to anon using (true) with check (true);
```

---

## 3. The 10 Active Prediction Questions (Sync with `index.html`)

All 10 questions are structured as 4-option multiple choice for exact-match grading (1 point per correct answer):

| # | Question ID | Question Text | The 4 Options |
|---|-------------|---------------|---------------|
| **1** | `cry_first` | Who will cry first? | `Bride's Mom`, `Bride's Dad`, `Groom's Mom`, `Groom's Dad` |
| **2** | `speech_len` | How long will the best friend's speech be? | `Under 2 min`, `2 to 3 min`, `3 to 5 min`, `5+ min (grab a drink)` |
| **3** | `courses` | How many courses will dinner have? | `3 courses`, `4 courses`, `5 courses`, `6+ courses feast` |
| **4** | `fruit` | How many pieces of fruit in the venue? | `200 to 400`, `401 to 600`, `601 to 800`, `800+ pieces` |
| **5** | `dinner_games` | How many games will we play in the dinner reception? | `2 games`, `3 games`, `4 games`, `5 games` |
| **6** | `indo_songs` | How many Indo songs will the band sing in the dinner reception? | `4 songs`, `5 songs`, `6 songs`, `7 songs` |
| **7** | `last_person_out` | At what time will the last person leave after the party? | `11:30 PM`, `12:00 AM`, `12:30 AM`, `1:00 AM` |
| **8** | `after_party_food` | What will be the after-party food? | `KFC`, `McDonald's`, `Sate Taichan`, `Indomie` |
| **9** | `blackout_first` | From which group will someone blackout first? | `Canisius boys`, `Bride's cousins`, `Perth crew`, `Groom himself` |
| **10** | `love_first` | Who said 'I love you' first? | `Jesson for sure`, `Diorra without hesitation`, `Said it at the same exact time`, `Still debating who did!` |

---

## 4. Admin Dashboard Key Features & Workflows

### A. Prediction Settlement Console (Tab 1)
- **Interactive Question Cards:** Displays all 10 questions with their 4 option chips.
- **One-Tap Resolution:** Clicking any option immediately upserts `question_answers` in Supabase.
- **Visual Feedback:** 
  - Selected correct answer glows in gold badge with `✓ Settled`.
  - Shows total bets submitted for this question and a breakdown list of guests who guessed right vs. wrong.
- **Clear/Unsettle Button:** Option to reset a question answer if marked by mistake.

### B. MC Master Leaderboard (Tab 2)
- **Automated Score Tallying:** Computes:
  $$\text{Score}(\text{Guest}) = \sum \mathbb{I}(\text{Guest's Answer} = \text{Correct Answer})$$
  *(Ties at top rank are highlighted gracefully side-by-side).*
- **Leaderboard Podium UI:** 
  - 🥇 **1st Place:** Grand Prediction Prize Winner ($N/10$ correct).
  - 🥈 **2nd Place** & 🥉 **3rd Place** runner-up cards.
  - Complete ranking list of all participating dinner guests with their individual picks expandable.
- **MC Screen Mode / Big Screen Toggle:** High-contrast full-screen view suitable for projection or MC iPad reading during prize handover.

### C. Photo Hunt Claims Tally (Tab 3)
- **Realtime Photo Leaderboard:** Aggregates entries from `photo_claims` grouped by `guest_name`.
- **Rank by Unique Categories Claimed:** Shows who captured the most categories (out of 12 active hunt categories).
- **Direct Drive Link:** Quick button to open the Google Drive folder (`1rSQcuy3yNZgb6IzYOsBMlisbXov4YGLZ`) to review the actual uploaded photos.

### D. Emergency Controls & Tools (Tab 4)
- **Market Lock Toggle:** One-switch control to lock/unlock guest prediction submissions from `index.html`.
- **Export Raw Data to CSV:** Downloads full timestamped `predictions.csv` and `photo_claims.csv` for keepsake or spreadsheet analysis.
- **Clear Test Submissions:** Safe button to wipe pre-event test submissions before the dinner begins.

---

## 5. Design & User Experience Guidelines

- **Theme & Aesthetics:** Inherits the luxury palette from `index.html` (Deep Obsidian `#120e0c`, Champagne Gold `#c5a059`, Ivory Paper cards `#f9f5ed`).
- **Responsive Layout:**
  - Mobile-friendly for quick tapping by the couple on their phones.
  - Expands to multi-column widescreen on iPad/laptops for stage managers and MCs.
- **Offline & Low-Latency Tolerance:** Uses Supabase Realtime subscriptions + manual `↻ Refresh` button so it functions smoothly over ballroom Wi-Fi / 5G.

---

## 6. Implementation Checklist

1. [ ] Create Supabase tables `question_answers` and `market_settings` (SQL in Section 2).
2. [ ] Build `admin.html` with PIN protection (`1003`), settlement buttons, MC leaderboard, and photo claims counter.
3. [ ] Connect realtime subscription on `predictions` and `question_answers`.
4. [ ] Test settling questions and verify leaderboard calculations live.
5. [ ] Deploy to Netlify.
