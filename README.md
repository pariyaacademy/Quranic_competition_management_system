# Musabaqa — Qur'an Import & Project Status

## Why this script exists (and why I didn't paste the Qur'an text through chat)

Loading 6,236 ayahs (~1.3MB of Arabic text) through this chat's tool-call
interface risks silent truncation — the exact kind of invisible data
corruption the project spec explicitly forbids for Qur'anic text. So instead
of retyping/relaying the verse text myself, this script reads your local
Tanzil file directly and streams it into Supabase over a real database
connection, with exhaustive validation before a single row is written.

## What's already live in your Supabase project (`pariyaacademy's Project`)

- New `musabaqa` Postgres schema, fully separate from your existing school
  tables (`public.students`, `public.results`, etc.) — nothing here touches
  or depends on the school system.
- `musabaqa.app_role` enum + `musabaqa.user_roles` table (independent of
  `public.profiles.role`, which stays untouched).
- `musabaqa.quran_sources`, `musabaqa.surahs` (all 114, verified ayah counts),
  and an empty `musabaqa.ayahs` table — RLS enabled, public read-only,
  no UPDATE policy at all (so the text is structurally immutable — not even
  a compromised admin session can edit it after import).

## Running the import

1. Install dependencies:
   ```bash
   npm install
   ```
2. Set your Supabase credentials (find these in Project Settings → API in
   the Supabase dashboard for `pariyaacademy's Project`):
   ```bash
   export SUPABASE_URL="https://dcvgdmtkuccfmsbosboc.supabase.co"
   export SUPABASE_SERVICE_ROLE_KEY="..."   # service_role key — keep secret, never commit
   ```
3. Run the importer against your validated Uthmani file:
   ```bash
   npm run import:quran -- /path/to/quran-uthmani.txt
   ```

The script will:
- Refuse to run if the file's embedded Tanzil copyright block doesn't say
  "Uthmani" (guards against accidentally importing the wrong edition again).
- Re-validate structure from scratch (114 surahs, 6236 ayahs, no duplicates,
  no gaps, no count mismatches) and print the same validation report format
  specified in the project brief.
- Abort with no writes at all if validation fails.
- Skip re-importing if this exact file (by checksum) was already imported.
- Insert a `quran_sources` row, then batch-insert all ayahs referencing it.

Afterwards, run `npm run validate:quran` any time to re-check the live
database (surah/ayah counts) without touching the source file.

## Next steps (not yet built)

- Structural metadata columns on `ayahs` — `juz_number`, `hizb_number`,
  `rub_hizb_number`, `page_number`, `ruku_number`, `manzil_number`, `sajdah`
  — are currently NULL. These need Tanzil's `quran-data.xml` (or an
  equivalent verified boundary dataset), which hasn't been supplied yet.
  I did not fabricate these values, since a single wrong Juz/Hizb boundary
  would silently corrupt which questions fall inside a "10 Juz" category
  and similar competition rules. Upload that file when ready and I'll wire
  it in the same careful way.
- Competition core tables (competitions, categories, participants, judges,
  question engine, sessions, scoring, results, certificates) — schema design
  is ready per the architecture discussed; not yet applied to the database.
- RBAC policies beyond the Qur'an tables.
- The actual Next.js application.
