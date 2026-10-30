<div align="center">

# Indian Startup Funding: Data Cleaning & Preparation

**SWYNEX Technologies · Data Analyst Internship · Task 1**

![Python](https://img.shields.io/badge/Python-3.13-3776AB?logo=python&logoColor=white)
![pandas](https://img.shields.io/badge/pandas-Data%20Cleaning-150458?logo=pandas&logoColor=white)
![Google Colab](https://img.shields.io/badge/Google%20Colab-Notebook-F9AB00?logo=googlecolab&logoColor=white)
![Status](https://img.shields.io/badge/Status-Completed-2EA44F)

*A raw, messy startup-funding dataset cleaned into an analysis-ready table.*

</div>

---

## Table of Contents
1. [Project Overview](#1-project-overview)
2. [Dataset](#2-dataset)
3. [Repository Structure](#3-repository-structure)
4. [Data Quality Audit (Before Cleaning)](#4-data-quality-audit-before-cleaning)
5. [Cleaning Workflow](#5-cleaning-workflow)
6. [Before vs After](#6-before-vs-after)
7. [Final Data Dictionary](#7-final-data-dictionary)
8. [Decisions & Assumptions](#8-decisions--assumptions)
9. [Limitations](#9-limitations)
10. [How to Reproduce](#10-how-to-reproduce)
11. [Key Learnings](#11-key-learnings)
12. [Author](#12-author)

---

## 1. Project Overview

**Objective:** Choose a public dataset, identify data-quality problems (missing values, duplicate records, incorrect data types, inconsistent values) and clean it using Python so it is ready for analysis.

**Approach:** Every step was done in a Google Colab notebook with `pandas` and regular expressions. Each problem was first **detected** (with a check), then **fixed**, then **re-verified**.

**Outcome:** A cleaned dataset of **3,044 rows × 10 columns** with correct data types, standardised categories, decoded text and clearly handled missing values.

---

## 2. Dataset

| Property | Detail |
|---|---|
| Source | Kaggle, Indian startup funding dataset |
| Raw file | `startup_funding.csv` |
| Size | 3,044 rows × 10 columns |
| Period covered | January 2015 to January 2020 |
| Content | Funding rounds of Indian startups: date, company, industry, city, investors, round type, amount in USD |

---

## 3. Repository Structure

```
SWYNEX-Data-Cleaning-Preparation/
├── startup_funding.csv                      # Original raw data (untouched)
├── cleaned_startup_funding.csv              # Final cleaned data
├── Indian_Startup_Funding_Data_Cleaning_&_Preparation.ipynb   # Full notebook with every step
└── README.md                                # This document
```

---

## 4. Data Quality Audit (Before Cleaning)

Checked using `df.shape`, `df.info()`, `df.isna().sum()` and `df.duplicated().sum()`.

| # | Problem found | Where | Scale |
|---|---|---|---|
| 1 | **Missing values** | Remarks, Amount, SubVertical, City, Industry, Investors, Investment Type | Remarks 2,625 · Amount 960 · SubVertical 936 · City 180 · Industry 171 · Investors 24 · Type 4 |
| 2 | **Wrong data type** | `Date` stored as text | all 3,044 rows |
| 3 | **Wrong data type** | `Amount in USD` stored as text (Indian commas `20,00,00,000`, `+`, `undisclosed`, `N/A`) | 2,084 non-null values |
| 4 | **Malformed dates** | `05/072018`, `01/07/015`, `12/05.2015`, `22/01//2015`, hidden characters | 8 rows |
| 5 | **Inconsistent city names** | Bangalore / Bengaluru, Gurgaon / Gurugram, Delhi / New Delhi, typos, two cities in one cell (`Pune / US`) | 112 raw labels |
| 6 | **Inconsistent investment types** | `Seed Funding`, `Seed/ Angel Funding`, `Seed / Angle Funding`, `PrivateEquity`, case differences | 50+ variants |
| 7 | **Inconsistent industry labels** | `eCommerce`, `ECommerce`, `E-Commerce`, `Ecommerce`, `ecommerce` | 821 unique labels |
| 8 | **Hidden encoding junk** | `\xe2\x80\x99`, `\xc2\xa0`, `\n`, `\xc3\x98` written as literal text | 27 startup names, 92 investor entries |
| 9 | **URL used as a name** | `https://www.wealthbucket.in/` in `Startup Name` | 1 row |
| 10 | **Unusable column headers** | `City  Location` (double space), `InvestmentnType` (typo), `Date dd/mm/yyyy` | 3 columns |
| 11 | **Duplicates** | Full-row duplicate check | **0 found** |

---

## 5. Cleaning Workflow

### Step 1: Load the data
```python
import pandas as pd
df = pd.read_csv('/content/drive/MyDrive/startup_funding.csv', encoding='utf-8-sig')
```
`utf-8-sig` removes a hidden byte-order marker that otherwise corrupts the first column name (`ï»¿Sr No`).

### Step 2: Standardise column names
```python
df.columns = ['sr_no','date','startup_name','industry','subvertical',
              'city','investors','investment_type','amount_usd','remarks']
```
Short `snake_case` names, no spaces, no typos.

### Step 3: Fix the `date` column
Eight dates were malformed, so the repairs were done with regular expressions instead of manual edits:
```python
d = df['date'].str.replace(r'\\+xc2\\+xa0', '', regex=True)        # hidden junk
d = d.str.replace(r'[./]+', '/', regex=True)                       # '12/05.2015', '22/01//2015'
d = d.str.replace(r'^(\d{2}/\d{2})(\d{4})$', r'\1/\2', regex=True) # '05/072018' -> '05/07/2018'
d = d.str.replace(r'/0(\d{2})$', r'/20\1', regex=True)             # '01/07/015' -> '01/07/2015'
df['date'] = pd.to_datetime(d, format='%d/%m/%Y')
```
Result: `object` → `datetime64[ns]`, zero unparseable dates.

### Step 4: Convert `amount_usd` to numeric
```python
a = df['amount_usd'].str.replace(r'\\+xc2\\+xa0', '', regex=True)
a = a.str.replace(r'[,+]', '', regex=True)
df['amount_usd'] = pd.to_numeric(a, errors='coerce')
```
Commas and `+` removed; non-numeric values (`undisclosed`, `unknown`, `N/A`) become `NaN`. Result: `object` → `float64`.

### Step 5: Standardise `city`
```python
c = df['city'].str.replace(r'\\+xc2\\+xa0', '', regex=True).str.strip()
c = c.str.split(r'\s*/\s*', regex=True).str[0]      # keep the first city
c = c.replace({'Bangalore':'Bengaluru', 'Gurgaon':'Gurugram', 'Delhi':'New Delhi',
               'Ahemadabad':'Ahmedabad', 'Trivandrum':'Thiruvananthapuram'})
df['city'] = c.fillna('Unknown')
```
Multi-city cells keep the first (primary) city; spelling variants are merged; missing becomes `Unknown`.

### Step 6: Group `investment_type` into clean categories
A small function maps 50+ raw labels into 16 categories using lowercase keyword checks (`seed`/`angel` → Seed/Angel, `debt`/`loan` → Debt, `series a`…`series j`, `pre-series a`, `private`/`equity` → Private Equity, `crowd` → Crowdfunding, otherwise Other / Unknown).

| Final category | Rows |
|---|---|
| Seed/Angel | 1,542 |
| Private Equity | 1,367 |
| Debt | 31 |
| Series A / B / C / D | 24 / 21 / 14 / 12 |
| Other | 11 |
| Pre-Series A | 9 |
| Unknown | 4 |
| Series E / F / G / H / J | 2 / 2 / 1 / 1 / 1 |
| Crowdfunding | 2 |

### Step 7: Merge common `industry` variants
```python
k = df['industry'].str.lower().str.replace(r'[^a-z]', '', regex=True)
mapping = {'ecommerce':'eCommerce', 'edtech':'EdTech', 'fintech':'FinTech',
           'foodbeverage':'Food & Beverage', 'foodandbeverage':'Food & Beverage',
           'foodbeverages':'Food & Beverage', 'foodandbeverages':'Food & Beverage',
           'it':'Technology', 'informationtechnology':'Technology',
           'transport':'Transportation', 'others':'Other'}
df['industry'] = k.map(mapping).fillna(df['industry']).fillna('Unknown')
```
Labels are lowercased and stripped to letters only so `E-Commerce`, `ECommerce` and `ecommerce` collapse to one key. Unique labels: **821 → 806**; `eCommerce` grew from 186 to **299** rows.

### Step 8: Decode hidden encoding junk
Special characters had been saved as literal text (`\xe2\x80\x99` instead of `’`). A reusable decoder converts the hex bytes back to real characters:
```python
import re

def decode_junk(m):
    hexes = re.findall(r'x([0-9a-fA-F]{2})', m.group())
    return bytes(int(h, 16) for h in hexes).decode('utf-8', errors='ignore')

for col in ['startup_name', 'subvertical', 'investors', 'remarks']:
    s = df[col].str.replace(r'(?:\\+x[0-9a-fA-F]{2})+', decode_junk, regex=True)
    s = s.str.replace(r'\\+n', ' ', regex=True)
    df[col] = s.str.replace(r'\s+', ' ', regex=True).str.strip()
```
Verification after the fix: **0** startup names and **0** investor entries with junk remaining (earlier: 27 and 92).

### Step 9: Handle missing text values
```python
df['investors']   = df['investors'].fillna('Undisclosed')
df['subvertical'] = df['subvertical'].fillna('Not provided')
df['remarks']     = df['remarks'].fillna('Not provided')
```

### Step 10: Clean URL-style startup name
```python
df['startup_name'] = (df['startup_name']
                      .str.replace(r'^https?://(www\.)?', '', regex=True)
                      .str.rstrip('/'))
```
`https://www.wealthbucket.in/` → `wealthbucket.in`.

### Step 11: Final validation and export
```python
df.info()
print(df.duplicated().sum())          # 0
df.to_csv('cleaned_startup_funding.csv', index=False)
```

---

## 6. Before vs After

| Check | Before | After |
|---|---|---|
| Shape | 3,044 × 10 | 3,044 × 10 |
| `date` type | text | `datetime64[ns]` |
| `amount_usd` type | text | `float64` |
| Malformed dates | 8 | 0 |
| Missing: investors | 24 | 0 (`Undisclosed`) |
| Missing: subvertical | 936 | 0 (`Not provided`) |
| Missing: remarks | 2,625 | 0 (`Not provided`) |
| Missing: city | 180 | 0 (`Unknown`) |
| Missing: industry | 171 | 0 (`Unknown`) |
| Missing: investment type | 4 | 0 (`Unknown`) |
| Missing: amount | 960 | 971 (see note below) |
| Investment type labels | 50+ | 16 |
| Industry labels | 821 | 806 |
| Encoding junk (names / investors) | 27 / 92 | 0 / 0 |
| Duplicate rows | 0 | 0 |

> **Why amount missing went from 960 to 971:** 11 values such as `undisclosed`, `unknown` and `N/A` were text, not blanks. Once the column became numeric they were correctly treated as missing.

---

## 7. Final Data Dictionary

| Column | Type | Description |
|---|---|---|
| `sr_no` | int | Original serial number |
| `date` | datetime | Date of the funding round |
| `startup_name` | text | Company that received funding |
| `industry` | text | Industry vertical (common variants merged) |
| `subvertical` | text | More specific business description (`Not provided` if blank) |
| `city` | text | Primary city of the startup (`Unknown` if blank) |
| `investors` | text | Investor name(s) (`Undisclosed` if blank) |
| `investment_type` | text | Standardised round type (16 categories) |
| `amount_usd` | float | Funding amount in USD (`NaN` if undisclosed) |
| `remarks` | text | Extra notes (`Not provided` if blank) |

---

## 8. Decisions & Assumptions

- **Amount was not imputed.** An undisclosed funding amount is a real fact, not a data error. Filling it with a mean or median would invent money that was never raised and distort any later analysis.
- **First city is kept** when two are listed, as it is treated as the startup's primary location.
- **`Delhi` was merged into `New Delhi`**, and `Gurgaon` into `Gurugram`, as the same locations under different names.
- **Original file is preserved** unchanged, so every step can be audited and reproduced.
- **Duplicate check** was run on complete rows; none were found, so no rows were dropped.

---

## 9. Limitations

- `industry` is free text in the source, so about **800 labels remain**. Only the high-frequency variants were merged; the long tail of single-occurrence labels was left as is.
- `amount_usd` remains missing for 971 rows, as the amount was never disclosed.
- Cities listed second in multi-city cells (for example `US` in `Pune / US`) are dropped.
- A few investor names with accented letters (for example `Ørstead`) were decoded automatically but not manually verified one by one.

---

## 10. How to Reproduce

1. Clone this repository.
2. Open `Indian_Startup_Funding_Data_Cleaning_&_Preparation.ipynb` in Google Colab or Jupyter.
3. Update the file path in Step 1 to where `startup_funding.csv` is stored.
4. Run all cells (**Runtime → Run all**).
5. The cleaned file is written as `cleaned_startup_funding.csv`.

**Requirements:** Python 3, `pandas`.

---

## 11. Key Learnings

- Detect first, fix second, verify third: every cleaning step had a before-and-after check.
- Regular expressions handle patterns of errors in one pass, instead of editing cells one by one.
- Text can hide encoding problems (`\xe2\x80\x99` = `’`) that look fine until you search or group by that column.
- Not every missing value should be filled; the right treatment depends on what the blank means.
- Cleaning decisions (what to merge, what to keep) are judgement calls and should be documented.

---

## 12. Author

**Sushant Kumar Yadav**
Aspiring Data Analyst · B.Com (Hons), University of Delhi

LinkedIn: [www.linkedin.com/in/sushantkumaryadav310899](https://www.linkedin.com/in/sushantkumaryadav310899/)
GitHub: [Skysushant7366](https://github.com/Skysushant7366)
Portfolio: [https://skysushant7366.github.io/Sushant-portfolio/](https://skysushant7366.github.io/Sushant-portfolio/)
*Completed as Task 1 of the Data Analyst Internship at SWYNEX Technologies.*
