# 💰 Expense Tracker — Project Plan

> A Flask web app that turns everyday expenses into clear insights, budget alerts and
> (later) smart predictions — built for how people in India actually spend (UPI, cash, cards).

---

## 1. Goal

**Problem:** Most people don't know where their money goes each month. UPI makes spending
fast and invisible, and bank statements are hard to read.

**Solution:** A simple app where I log expenses in seconds and instantly see:
1. **Where** my money is going
2. **Whether** I'm spending more or less than before
3. **If** I'm within my budget
4. *(Later)* **What** I'm likely to spend by month-end

**Learning goals:** Flask, SQL, Pandas, Chart.js, testing, deployment — and later, ML.

---

## 2. Data Model

### 2.1 `expenses` table

| Field | Type | Required | Notes |
|---|---|---|---|
| `id` | INTEGER (primary key, auto-increment) | ✅ | Unique ID for edit/delete |
| `date` | TEXT (`YYYY-MM-DD`) | ✅ | ISO format sorts correctly as text |
| `amount` | REAL | ✅ | Decimals allowed (₹99.50) |
| `category` | TEXT | ✅ | Must be from the fixed list |
| `payment_method` | TEXT | ✅ | Must be from the fixed list |
| `description` | TEXT | ❌ | Short note, e.g. "Lunch with team" |
| `tags` | TEXT | ❌ | Comma-separated, e.g. "trip-goa,friends" *(Phase 2)* |
| `is_recurring` | INTEGER (0/1) | ❌ | Created from a recurring rule *(Phase 2)* |
| `source` | TEXT | ✅ | `manual` / `quick_add` / `csv` / `statement` — where the row came from. Default `manual`. |
| `raw_description` | TEXT | ❌ | Original bank text, e.g. `UPI/SWIGGY/4567/PAYMENT`, kept untouched for ML training |
| `transaction_ref` | TEXT | ❌ | Bank reference / UTR number — used to **detect duplicates** when the same statement is imported twice |
| `import_batch_id` | INTEGER | ❌ | Links the row to the import it came from, so a bad import can be undone in one click |
| `created_at` | TEXT (timestamp) | ✅ | Auto-filled when saved |

> 💡 **Why add the import columns now?** Adding columns to a table that already has data
> needs a migration. Adding them on day one costs nothing — they simply stay empty for
> manually entered expenses.

### 2.2 `budgets` table

| Field | Type | Notes |
|---|---|---|
| `category` | TEXT (primary key) | One budget per category |
| `monthly_limit` | REAL | Must be ≥ 0 |

### 2.3 `import_batches` table (for bank statement automation)

| Field | Type | Notes |
|---|---|---|
| `id` | INTEGER (primary key) | |
| `file_name` | TEXT | e.g. `HDFC_Oct_2026.csv` |
| `bank` | TEXT | Which bank format was detected |
| `imported_at` | TEXT (timestamp) | |
| `rows_found` | INTEGER | Transactions in the file |
| `rows_imported` | INTEGER | Actually saved |
| `rows_skipped` | INTEGER | Duplicates, credits/income, or invalid rows |

### 2.4 `category_rules` table (keyword → category, learns from my corrections)

| Field | Type | Notes |
|---|---|---|
| `keyword` | TEXT (primary key) | e.g. `swiggy`, `uber`, `jio` |
| `category` | TEXT | e.g. Food, Transport, Bills |

### 2.5 Later tables (Phase 2+)

- **`recurring_expenses`** — rent, subscriptions, EMIs: `id, name, amount, category, payment_method, day_of_month, active`
- **`income`** — salary, stipend, freelance: `id, date, amount, source, description`
- **`savings_goals`** — `id, name, target_amount, saved_amount, deadline`

---

## 3. Fixed Lists

### Categories
1. Food & Dining
2. Groceries
3. Transport (auto, metro, cab, fuel)
4. Rent & Housing
5. Bills & Utilities (electricity, mobile, internet)
6. Shopping
7. Entertainment & Subscriptions
8. Health & Fitness
9. Education
10. Travel
11. Gifts & Festivals
12. Other

### Payment methods
- UPI
- Cash
- Debit Card
- Credit Card
- Net Banking
- Wallet (Paytm, Amazon Pay, etc.)

---

## 4. Validation Rules

| Rule | Decision | Reason |
|---|---|---|
| Amount must be > 0 | ✅ Enforced | A ₹0 or negative expense makes totals wrong |
| Maximum amount | ₹10,00,000 | Catches typos like an extra zero |
| Future dates | ❌ Not allowed | An expense is something that already happened. Planned costs go in Budgets. |
| Very old dates | Max 5 years back | Catches wrong-year typos |
| Category / payment method | Must match the fixed list | Keeps charts and grouping clean |
| Description | Optional, max 200 characters | Faster entry, but still allows context |
| Validation location | **Server-side** (and browser-side for convenience) | Browser checks can be bypassed |

---

## 5. Pages & Routes

| Page | Route | Method | User sees | User can do |
|---|---|---|---|---|
| Home / Dashboard | `/` | GET | KPIs, charts, insights, budget alerts | Pick a period (7d / 30d / month / all / custom) |
| Add Expense | `/add` | GET, POST | Form + "recent 5 expenses" list | Save an expense, use Quick Add |
| All Expenses | `/expenses` | GET | Table with filters, count, total | Filter, search, sort, paginate, export |
| Edit Expense | `/expenses/<id>/edit` | GET, POST | Pre-filled form | Update the expense |
| Delete Expense | `/expenses/<id>/delete` | POST only | Confirmation prompt | Delete the expense |
| Export CSV | `/expenses/export` | GET | — | Download filtered expenses |
| Import CSV | `/import` | GET, POST | Upload form + preview | Bulk-add expenses |
| Import Statement | `/import/statement` | GET, POST | Upload bank statement, pick bank | Upload file → go to review |
| Review Import | `/import/review/<batch_id>` | GET, POST | Table of parsed transactions with suggested categories | Fix categories, untick rows, confirm import |
| Import History | `/import/history` | GET | List of past imports | Undo (delete) a whole import |
| Budgets | `/budgets` | GET, POST | Progress bars per category | Set or change limits |
| JSON API | `/api/summary?period=30d` | GET | JSON | Used by the dashboard charts |
| 404 page | — | — | Friendly "not found" page | Go back home |

### Page details

**Dashboard (`/`)**
- KPI cards: Total spent · Transactions · Average per day · Top category · vs last month (%)
- Charts: category donut · daily trend line · weekday bars · payment-method split · month-over-month bars
- Insights section (auto-generated sentences)
- Budget warning banner if any category is ≥ 80%

**Add Expense (`/add`)**
- Date defaults to today
- Amount field gets focus automatically
- Quick Add bar (see Special Features)
- List of the last 5 expenses, so I can see it saved

**All Expenses (`/expenses`)**
- Filters live in the URL (`?category=Food&from=2026-10-01`) so they can be bookmarked
- Sort by date or amount
- Pagination (20 per page)
- Edit and Delete buttons on each row

**Budgets (`/budgets`)**
- 🟢 < 80% · 🟠 80–100% · 🔴 > 100%
- Shows "₹X left" or "Over by ₹X"
- "Safe to spend per day" for the rest of the month

---

## 6. ⭐ Special Features (what makes this app stand out)

### Phase 1 — Core-plus (build with the MVP)

1. **⚡ Quick Add (natural-language entry)**
   Type `150 chai upi` or `1200 uber card yesterday` and press Enter. The app parses the amount,
   guesses the category from keywords ("chai" → Food), the payment method and the date.
   *Learn:* string parsing, regex, keyword mapping.

2. **💡 Smart Insights in plain English**
   - "Food is your biggest expense: ₹7,800 (32% of total)."
   - "You spend 2× more on weekends than weekdays."
   - "Your largest expense this month was ₹3,200 on Shopping."
   - "You spent 18% more than last month."

3. **🗓️ Calendar Heatmap**
   A GitHub-style grid where each day is coloured by how much was spent. Instantly shows
   high-spend days and **no-spend days**.

4. **🔥 No-Spend Streak**
   "You've had 3 no-spend days this week. Current streak: 2 days."
   A small habit-building touch that makes the app feel personal.

5. **🎯 Safe-to-Spend Daily Limit**
   (Remaining budget ÷ days left in month) → "You can spend ₹420/day for the rest of November."

### Phase 2 — Power features

6. **🔁 Recurring Expenses**
   Set up rent, Netflix, gym or EMIs once; they're added automatically every month.

7. **📺 Subscription Tracker**
   A list of all recurring subscriptions with the yearly total:
   "Your subscriptions cost ₹7,188/year."

8. **🏷️ Tags for events**
   Tag expenses like `goa-trip` or `diwali-2026` and see the total cost of any trip or festival.

9. **💵 Income & Savings Rate**
   Log income and see: "You saved 22% of your income this month."

10. **🏦 Savings Goals**
    "New laptop: ₹32,000 / ₹60,000 (53%) — on track for March."

11. **📄 Monthly Report (PDF)**
    One click → a clean one-page PDF summary of the month with charts.

12. **🌙 Dark Mode + Mobile-friendly design**
    Responsive Bootstrap layout; installable on a phone home screen as a PWA.

### Phase 3 — AI / ML (connects to my ML learning)

13. **🤖 Auto-Categorization (ML)**
    Train a TF-IDF + Logistic Regression model on descriptions, so "Swiggy order" or
    "UPI/ZOMATO/123" is labelled Food automatically. Show the model's confidence.

14. **🏦 Bank Statement Import — ML upgrade**
    The rule-based importer (see section 7) gets smarter: the ML model from #13 predicts
    categories for descriptions that no keyword rule matches, and shows its confidence.

15. **📈 Month-End Forecast**
    "At your current pace, you'll spend ₹28,400 this month (budget: ₹25,000)."
    Start with a simple projection, then try linear regression.

16. **🚨 Unusual Expense Detection**
    Flag transactions far outside normal patterns using z-scores, then Isolation Forest:
    "₹5,000 on Food is unusual for you."

17. **💬 Ask Your Expenses (LLM)**
    Ask in plain English: "How much did I spend on cabs in October?"
    An LLM converts the question into a safe query on my data.

---

## 7. 🏦 Automated Data Entry from Bank Statements

**Goal:** stop typing every expense. Upload a monthly bank statement and the app adds all the
spending transactions for me, already categorized.

### 7.1 How it works (pipeline)

```
Upload file → Detect bank format → Parse rows → Clean → Keep only debits
   → Remove duplicates → Suggest category → Review screen → Save → Import history
```

1. **Upload** — CSV / Excel first; PDF later (with `pdfplumber`)
2. **Detect format** — each bank names its columns differently
   (`Narration` / `Description` / `Particulars`, `Withdrawal Amt` / `Debit`, date formats like
   `04/11/26` vs `04-Nov-2026`). Keep one small **column-mapping config per bank**.
3. **Parse & clean** — convert dates to `YYYY-MM-DD`, strip commas from amounts
   (`1,250.00` → `1250.0`), trim spaces
4. **Keep only debits** — credits (salary, refunds) are skipped now, and go to the `income`
   table once it exists
5. **Clean the merchant name** — `UPI/SWIGGY/4567/PAYMENT` → `Swiggy`
   (regex; keep the original in `raw_description`)
6. **Detect duplicates** — same `transaction_ref`, or same date + amount + description, is
   skipped
7. **Suggest a category**
   - Phase 1: keyword rules (`swiggy`, `zomato` → Food; `uber`, `ola`, `irctc` → Transport)
   - Phase 3: ML model for anything no rule matches
8. **Review screen** — I see every transaction before it's saved, can change categories and
   untick rows. **Nothing is saved without my confirmation.**
9. **Learn from corrections** — if I change "Blinkit" from Other → Groceries, save `blinkit` →
   Groceries in `category_rules`, so next month it's automatic
10. **Save + history** — all rows get `source = 'statement'` and an `import_batch_id`;
    a whole import can be undone in one click

### 7.2 Build order

| Step | What | Phase |
|---|---|---|
| A | Database ready: `source`, `raw_description`, `transaction_ref`, `import_batch_id` columns | **Phase 1 (now)** |
| B | Generic CSV import with a fixed template (`date, amount, category, …`) | Phase 1 |
| C | One real bank's CSV (my own bank) + review screen + duplicate check | Phase 2 |
| D | Keyword category rules + learning from corrections | Phase 2 |
| E | Import history + undo | Phase 2 |
| F | More banks, Excel and PDF support | Phase 2 |
| G | ML auto-categorization (feature #13) | Phase 3 |

### 7.3 Code structure for importers

```
importers/
├── __init__.py
├── base.py            # Common steps: clean, dedupe, keep debits, suggest category
├── bank_formats.py    # Column mapping + date format per bank
├── csv_importer.py    # Reads CSV / Excel into a standard DataFrame
├── pdf_importer.py    # Reads PDF tables (later)
└── categorizer.py     # Keyword rules now, ML model later
```

Every importer outputs the **same standard DataFrame** (`date, amount, raw_description,
transaction_ref`), so the review screen and database code never care which bank the file
came from. Adding a new bank = adding one entry in `bank_formats.py`.

### 7.4 Privacy & safety rules

- ❗ **Never commit real bank statements to GitHub.** Add `statements/`, `*.pdf`, `*.xlsx`
  and real CSVs to `.gitignore`.
- Create a **fake sample statement** (`sample_data/sample_statement.csv`) for tests and the
  live demo.
- Delete the uploaded file after parsing — only the extracted transactions are stored.
- On the public demo, disable uploads or use only the sample file.
- Limit upload size (e.g. 5 MB) and allowed file types.

### 7.5 Tests

- Sample statement → correct number of debits parsed
- `1,250.00` → `1250.0`; `04-Nov-2026` → `2026-11-04`
- Importing the same file twice → second import adds **0** rows
- `UPI/SWIGGY/4567` → suggested category Food
- Undo import → all rows from that batch are deleted

---

## 8. Project Structure

```
expense-tracker/
├── app.py                 # Flask routes only
├── database.py            # All SQL (CRUD functions)
├── analytics.py           # All Pandas calculations
├── quick_add.py           # Natural-language parser (Phase 1)
├── config.py              # Settings (secret key from env variable)
├── seed_data.py           # Generates realistic sample data
├── importers/             # Bank statement automation (see section 7)
│   ├── base.py
│   ├── bank_formats.py
│   ├── csv_importer.py
│   ├── pdf_importer.py
│   └── categorizer.py
├── sample_data/
│   └── sample_statement.csv   # FAKE statement for tests & demo
├── templates/
│   ├── base.html          # Navbar + layout, all pages extend this
│   ├── dashboard.html
│   ├── add.html
│   ├── expenses.html
│   ├── edit.html
│   ├── budgets.html
│   ├── import.html
│   ├── import_review.html
│   ├── import_history.html
│   └── 404.html
├── static/
│   ├── style.css
│   └── charts.js          # Chart.js code for the dashboard
├── tests/
│   ├── test_database.py
│   ├── test_analytics.py
│   ├── test_quick_add.py
│   ├── test_importers.py
│   └── test_routes.py
├── screenshots/           # For the README
├── requirements.txt
├── .gitignore
├── README.md
└── PLAN.md
```

---

## 9. Why Separate Files? (Separation of Concerns)

Each file has **one job**:
- `database.py` — **storing** data
- `analytics.py` — **calculating** insights
- `app.py` — **talking to the user**

Benefits:
- **Easy to change:** switching SQLite → PostgreSQL only changes `database.py`.
  Switching Flask → Django or FastAPI reuses `database.py` and `analytics.py` unchanged.
- **Easy to test:** `analytics.py` functions can be tested with a small DataFrame — no
  browser or server needed.
- **Easy to debug:** a wrong total is a bug in `analytics.py`; a broken page is in `app.py`.
- **Easy to read:** small, focused files instead of one 800-line file.

---

## 10. Tech Stack

| Layer | Tool | Why |
|---|---|---|
| Backend | Flask | Small, teaches web fundamentals clearly |
| Templates | Jinja2 | Comes with Flask |
| Database | SQLite → PostgreSQL later | Zero setup to start |
| Analysis | Pandas | Industry-standard data tool |
| Charts | Chart.js | Simple, good-looking, free |
| Styling | Bootstrap 5 | Responsive design without much CSS |
| Testing | Pytest + Flask test client | Standard Python testing |
| Deployment | Render + Gunicorn | Free tier, deploys from GitHub |
| ML (Phase 3) | scikit-learn | Classic ML library |

---

## 11. Testing Plan

- **Database:** add → read → update → delete round-trip, using a temporary database
- **Validation:** reject amount ≤ 0, future dates, unknown categories
- **Analytics:** known small DataFrame → check totals, top category, month-over-month %
- **Empty data:** every analytics function works when there are no expenses
- **Quick Add:** `"150 chai upi"` → amount 150, Food, UPI, today
- **Statement import:** see section 7.5
- **Routes:** each page returns 200; invalid expense ID returns 404; POST redirects after saving

---

## 12. Milestones & Commit Plan

| # | Milestone | Done when… | Commit message |
|---|---|---|---|
| 0 | Setup | Flask "hello" runs, venv ignored | `Set up project structure and Flask` |
| 1 | Plan | This file is complete | `Add project plan` |
| 2 | Database | CRUD works from a Python shell | `Add database layer with CRUD functions` |
| 3 | Layout | Navbar + 4 empty pages share `base.html` | `Add base template and navigation` |
| 4 | Add expense | Form validates and saves | `Add expense form with validation` |
| 5 | Expenses list | Filter, edit, delete, export work | `Add expense list with filters, edit and delete` |
| 6 | Seed data | 3 months of realistic data | `Add sample data generator` |
| 7 | Analytics | All functions tested in a notebook | `Add analytics functions` |
| 8 | Dashboard | KPIs, charts and insights show | `Add analytics dashboard` |
| 9 | Budgets | Progress bars + warnings | `Add monthly budgets` |
| 10 | Special features | Quick Add, heatmap, streak, safe-to-spend | `Add Quick Add`, … (one commit each) |
| 11 | Tests | All tests pass | `Add tests` |
| 12 | Deploy | Live link works on a phone | `Deploy to Render` |
| 13 | README | Screenshots, demo link, "what I learned" | `Write README` |
| 14 | Statement import | Own bank's CSV → review → save, no duplicates | `Add bank statement import` |
| 15+ | Phase 2 & 3 | One feature at a time | One commit (or branch) per feature |

**Target:** MVP (milestones 0–13) in about 2–3 weeks. Phase 2 and Phase 3 afterwards.

---

## 13. Definition of Done (MVP)

- [ ] I can add, view, filter, edit, delete and export expenses
- [ ] Invalid input is rejected with clear messages
- [ ] Dashboard shows KPIs, at least 4 charts and auto insights
- [ ] Budgets show progress and warnings
- [ ] At least 2 special features from Phase 1 are working
- [ ] All tests pass
- [ ] App is live with a public link
- [ ] README has screenshots, a demo link and setup steps

---

## 14. Out of Scope (for now)

- User accounts / login → planned for the **Django version**
- Automatic SMS reading → requires an Android app
- Direct bank connection → requires a licensed Account Aggregator partner
  (statement upload in section 7 is the practical alternative)
- Multiple currencies
