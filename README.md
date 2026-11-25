# CareLoop Clinics: No-Show Guard

**Repository:** https://github.com/YOUR-USERNAME/careloop-noshow-guard  <!-- replace with your public repo link -->

**Business question:** *Which appointments are most likely to be missed, why do patients miss them, and what should CareLoop do first?*

From a raw booking export (Jan–Jun 2025, 8 clinics, 6 cities) to a cleaned table, SQL evidence of where and when no-shows cluster, a risk model scored the evening before each visit, and AI-extracted reasons from call-centre notes.

## Summary: one finding per Part

1. **Part 1 (data):** the 1,020-row raw export cleaned down to 990 unique appointments, of which 218 (22.0%) were **no-shows**.
2. **Part 2 (SQL):** bookings made **31+ days** ahead are missed **60.5%** of the time, against **16.2%** for bookings 1–7 days ahead.
3. **Part 3 (model):** logistic regression scores test ROC-AUC **0.789** and, at threshold **0.35**, catches **57.1%** of no-shows by flagging 49 of 198 bookings.
4. **Part 4 (GenAI):** all 9 non-blank notes were classified validly, and **both** "forgot / no reminder" notes (2 of 2) came from bookings that got **no reminder**.

## Recommendation

After carring out the model, my recommendation is to fix the reminders first, then phone the riskiest bookings each evening because reminders are the cheapest lever and the strongest protective signal in the data. In the model, a reminder goes with about **69% lower odds** of a no-show (odds × 0.31). Each of the 10 call-center notes belong to 1 missed appointment. For each appointment, the data records whether a reminder was set. In that case, both call-centre notes where the patient simply forgot does actually belong to bookings that got "no reminder". This pattern is visible across 5 of the 10 notes overall. Therefore, in my opinion, CareLoop should make sure every booking gets a reminder, starting with the clinics above the 22.0% average (Hadapsar 27.3%, Malviya Nagar 26.8%, Gomti Nagar 33.3%, Kothrud 22.5%). Moreover, the reception should run the logistic-regression score every evening and phone bookings at or above 0.35. On the test set that is about 1 booking in 4 (49 of 198) and catches 57% of the real no-shows (precision 0.57), at a cost of a minute's call against an empty doctor slot. Thirdly, bookings made far ahead require a reconfirmation call due to the no-shows rise from 16.2% (1–7 days) to 30.0% (15–30 days) and 60.5% (31+ days). Apart from this, the notes also point to transport, fees and work shifts as reasons, so evening or Saturday slots and a follow-up fee discount are worth giving a try. 

## Model chosen and why

- **Model:** logistic regression (`StandardScaler` + `LogisticRegression(max_iter=1000)` in a pipeline).
- **Why:** it had the highest 5-fold cross-validated ROC-AUC on the training set (0.774), ahead of the tuned random forest (0.767, `max_depth=3, n_estimators=200`), the default random forest (0.747) and the depth-3 tree (0.678). The choice used training data only. It is also interpretable: the front desk can see that long lead times, prior no-shows and no reminder push the risk up.
- **Threshold:** **0.35**, chosen from out-of-fold training probabilities. At 0.35, F1 = 0.489 and recall = 0.467 (154 flagged); at 0.5, F1 = 0.347 and recall = 0.249 (73 flagged). Recall matters most because a missed warning costs an empty slot, while an extra call costs a minute.
- **One test-set evaluation:** ROC-AUC **0.789**, precision 0.571, recall 0.571, **49 flagged** (see `test_risk_scores.csv`, 198 rows).
- In practice the threshold should be set by how many calls the front desk can make in an evening, weighed against the cost of an empty slot.

## Part 4 run notes

- **Model:** `gemini-3.5-flash-lite` (free tier), JSON mode with a response schema, temperature 0, thinking level `minimal`, 13-second pause between calls. Every reply was validated with Pydantic plus the allowed-list check.
- **Prompt choice (Task 4.3b):** without the allowed list, V1 made up its own labels for both test notes (a full sentence for N007, "Patient unreachable" for N009), so both failed the allowed-list check. V2 (the CRAFT prompt) returned `forgot_or_no_reminder` and `other`, both allowed, so **V2 is the one to deploy**.
- **API calls: 17 in total** (below 25): 1 in Task 4.1, 9 in Task 4.2, 2 in Task 4.3, plus 5 automatic retries. On the day of the final run Gemini returned 503 "model experiencing high demand" errors, so `call_gemini` waits and retries up to 3 times on server (5xx) errors only. Every attempt is counted. Retries never change or repair a reply; they only re-send a request that got no reply. Each API call also has a timeout, so a stalled request raises an error instead of hanging.

## Limitations

1. **The 10 notes are an illustration only.** They are the first 10 of 30 (free-tier limit) and cannot show the real reasons for no-shows clinic-wide.
2. **The cleaning statistics came from all rows before the split.** The Task 1.3 medians, quartiles and outlier fences were computed on all 990 rows, so the test rows had a small indirect influence on the cleaning; a real system would compute them on the training rows only.
3. **The data are observational, so associations are not causes.** The ~69% lower odds with a reminder does not prove that reminders cause fewer no-shows; for example, reminders may go more often to patients who would attend anyway.

## How to reproduce

| Step | File | Run in |
|---|---|---|
| Part 1: read, clean, enrich, export (Tasks 1.1–1.6) + Task 2.2 pandas check | `01_ingest_clean.ipynb` | Google Colab (upload `data/`) |
| Part 2: create tables, run `load_data.sql`, run queries | `02_queries.sql` | MySQL Workbench (MySQL 8) or SQLiteOnline |
| Part 3: model, tuning, threshold, test scores | `03_model.ipynb` | Colab (upload `appointments_clean.csv`) |
| Part 4: Gemini classification, Pydantic validation, crosstab | `04_genai.ipynb` | Colab (key typed at run time, never saved) |

Conventions: `random_state=42`, `test_size=0.2`, percentages rounded to 1 decimal, `np.std` (population). Reference run: scikit-learn 1.9.1, pandas 3.0.5, NumPy 2.5.3. All acceptance counts reproduced exactly. Part 1 was re-run in Colab so `clinics_enriched.csv` holds live Open-Meteo temperatures (all 8 calls returned status 200).

## Files

| File | What it is |
|---|---|
| `data/appointments.csv`, `data/clinics.csv`, `data/followup_notes.csv` | Original data pack (unchanged) |
| `rejected_rows.json` | 32 rows that failed type conversion in Task 1.1 (12 blank age, 20 blank distance) |
| `appointments_clean.csv` | 990 rows × 16 columns, the cleaned table used by Parts 2 and 3 |
| `clinics_enriched.csv` | 8 clinics + `temp_c` from Open-Meteo (first hourly reading, midnight UTC) |
| `load_data.sql` | 998 INSERT statements (8 clinics first, then 990 appointments) |
| `02_queries.sql` | All Part 2 queries, with results and the duplicate-key error as comments |
| `test_risk_scores.csv` | 198 test rows: appointment_id, no_show, risk_prob, flag (≥ 0.35) |
| `notes_analysed.csv` | 10 notes → root_cause, confidence, summary, recommended_action, validation_status, raw_response (9 valid, 1 skipped_blank) |

No API key is stored anywhere in this repository.
