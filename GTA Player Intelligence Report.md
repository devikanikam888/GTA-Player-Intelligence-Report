# GTA Online: Player Intelligence Report
### A Revenue & Retention Risk Analysis for a Live-Service Game

![Tools](https://img.shields.io/badge/Tools-SQL%20%7C%20Python%20%7C%20Excel%20%7C%20Power%20BI-blue)
![Records](https://img.shields.io/badge/Records-52%2C099%20Steam%20Reviews-green)
![Status](https://img.shields.io/badge/Status-Complete-brightgreen)

---

## Project Overview

This project was framed as a real-world business brief: acting as a Data Analyst hired by a gaming publisher to diagnose why a flagship live-service title is showing signs of player attrition and to identify the revenue risk before it becomes a crisis. Using 52,099 Steam reviews for GTA Online, the analysis moves through four tools in a deliberate pipeline — SQL for data exploration and segmentation, Python for NLP and sentiment intelligence, Excel for business modelling, and Power BI for interactive decision-making — with each tool building directly on the last. The goal was never to describe what the data shows but to answer what it means commercially and what should be done about it.

---

## Business Questions

| # | Question | Tool |
|---|---|---|
| 1 | Is player sentiment declining and when did it start? | SQL + Python + Power BI |
| 2 | Are Rockstar's highest-invested players becoming their loudest critics? | SQL + Python |
| 3 | When a player leaves a negative review are they actually gone or still logging in? | SQL + Python + Power BI |
| 4 | Are first-time reviewers being created at scale — people so frustrated they broke their silence? | SQL + Python + NLP |
| 5 | Is sentiment collapsing in specific player segments faster than others? | SQL + Excel + Power BI |
| 6 | Are early paying players the most abandoned segment? | SQL + Excel |
| 7 | What should the business do, who should they target first and what is the cost of doing nothing? | Excel + Power BI |

---

## Dataset

**Source:** Steam Reviews — GTA Online  
**Records:** 52,099 reviews  
**Date Range:** February 2024 onwards (hypothetical extended dataset)  
**Original Columns:** 16  
**Final Enriched Columns:** 26  

### Original Columns

| Column | Type | Description |
|---|---|---|
| id | BIGINT | Unique review identifier |
| language | VARCHAR | Review language |
| review | TEXT | Full review text |
| created | DATETIME | Date review was written |
| voted_up | BOOLEAN | Thumbs up (1) or thumbs down (0) |
| votes_up | INT | Number of community upvotes on the review |
| comment_count | INT | Number of comments on the review |
| steam_purchase | BOOLEAN | Whether reviewer purchased via Steam |
| recieved_for_free | BOOLEAN | Whether game was received for free |
| written_during_early_access | BOOLEAN | Whether written during early access period |
| author_num_games_owned | INT | Number of games owned by reviewer |
| author_num_reviews | INT | Total reviews written by this author |
| author_playtime_forever | INT | Total playtime in minutes |
| author_playtime_last_two_weeks | INT | Playtime in last two weeks in minutes |
| author_playtime_at_review | INT | Playtime at time of writing review in minutes |
| author_last_played | DATETIME | Last time the author played |

### Engineered Columns (added in Python)

| Column | Description |
|---|---|
| playtime_forever_hrs | author_playtime_forever converted to hours |
| playtime_at_review_hrs | author_playtime_at_review converted to hours |
| playtime_last_2weeks_hrs | author_playtime_last_two_weeks converted to hours |
| sentiment_score | VADER compound score (-1 to +1) |
| player_tier | Segmentation based on playtime: Casual / Moderate / Dedicated / Hardcore |
| reviewer_type | Segmentation based on review count: First Time / Occasional / Habitual |
| amplification_risk | Composite risk score combining votes, comments, playtime and reviewer credibility |
| player_segment | Paid Player / Free Player / Other based on steam_purchase and recieved_for_free |
| churn_status | Active / Churned / Frustrated based on voted_up and recent playtime |
| month | Month extracted from created for time series analysis |

---

## Tool Pipeline

```
Raw CSV → MySQL (SQL EDA) → Google Colab (Python NLP) → Excel (Business Model) → Power BI (Dashboard)
```

---

## Part 1 — SQL

### What and Why

SQL was the foundation of the entire project. The raw CSV was imported into a MySQL database after resolving encoding issues (utf8mb4 charset, pipe delimiter to handle review text containing commas). The first priority was never to analyse — it was to understand and validate. A clean view was created on top of the raw table so all analysis ran on verified, properly typed data while the original remained untouched. Ten EDA queries were written in logical order, each answering a specific business question, and seven views were saved so findings are permanently accessible to any team member without re-running queries. The business problems SQL solved: it established the 90.71% baseline, identified the U-shaped sentiment curve across player tiers, confirmed that 70.57% of negative reviewers had already churned, and pinpointed that every viral complaint pointed specifically at GTA Online infrastructure rather than the base game.

---

### Data Import

```sql
-- Step 1: Create database
CREATE DATABASE gta;
USE gta;

-- Step 2: Create table
CREATE TABLE gtadata (
    id BIGINT,
    language VARCHAR(50),
    review TEXT,
    created VARCHAR(50),
    voted_up BOOLEAN,
    votes_up INT,
    comment_count INT,
    steam_purchase BOOLEAN,
    recieved_for_free BOOLEAN,
    written_during_early_access BOOLEAN,
    author_num_games_owned INT,
    author_num_reviews INT,
    author_playtime_forever INT,
    author_playtime_last_two_weeks INT,
    author_playtime_at_review INT,
    author_last_played VARCHAR(50)
);

-- Step 3: Load data (pipe delimited after Python preprocessing)
LOAD DATA INFILE 'C:/ProgramData/MySQL/MySQL Server 8.0/Uploads/GTADATA.csv'
INTO TABLE gtadata
CHARACTER SET utf8mb4
FIELDS TERMINATED BY '|'
LINES TERMINATED BY '\n'
IGNORE 1 ROWS;
```

---

### Clean View

```sql
CREATE VIEW gta_clean AS
SELECT 
    id,
    language,
    review,
    STR_TO_DATE(created, '%d/%m/%Y %H:%i') AS created,
    voted_up,
    votes_up,
    comment_count,
    steam_purchase,
    recieved_for_free,
    written_during_early_access,
    author_num_games_owned,
    author_num_reviews,
    author_playtime_forever,
    author_playtime_last_two_weeks,
    author_playtime_at_review,
    STR_TO_DATE(author_last_played, '%d/%m/%Y %H:%i') AS author_last_played
FROM gtadata
WHERE id IS NOT NULL;
```

---

### EDA Query 1 — Data Health Check

```sql
SELECT 
    COUNT(*) AS total_reviews,
    COUNT(review) AS reviews_with_text,
    COUNT(*) - COUNT(review) AS empty_reviews,
    MIN(created) AS earliest_review,
    MAX(created) AS latest_review,
    DATEDIFF(MAX(created), MIN(created)) AS days_covered
FROM gta_clean;
```

**Finding:** 52,099 rows, 219 days of data, clean date range confirmed.

---

### EDA Query 2 — Sentiment Overview

```sql
SELECT 
    voted_up,
    COUNT(*) AS total,
    ROUND(COUNT(*) * 100.0 / SUM(COUNT(*)) OVER(), 2) AS percentage
FROM gta_clean
GROUP BY voted_up;
```

**Finding:** 90.71% positive, 9.29% negative. This becomes the baseline every subsequent metric is compared against.

---

### EDA Query 3 — Sentiment Over Time (Saved as View)

```sql
CREATE VIEW vw_sentiment_by_month AS
SELECT 
    DATE_FORMAT(created, '%Y-%m') AS month,
    COUNT(*) AS total_reviews,
    SUM(voted_up) AS positive,
    COUNT(*) - SUM(voted_up) AS negative,
    ROUND(SUM(voted_up) * 100.0 / COUNT(*), 2) AS positive_rate
FROM gta_clean
GROUP BY DATE_FORMAT(created, '%Y-%m')
ORDER BY month;
```

**Finding:** Opening month showed 78.79% positive — 12 points below baseline. First signal of structural decline.

---

### EDA Query 4 — Sentiment by Player Investment Tier (Saved as View)

```sql
CREATE VIEW vw_sentiment_by_playtime AS
SELECT 
    CASE 
        WHEN author_playtime_forever < 600 THEN '1. Casual (0-10hrs)'
        WHEN author_playtime_forever < 3000 THEN '2. Moderate (10-50hrs)'
        WHEN author_playtime_forever < 12000 THEN '3. Dedicated (50-200hrs)'
        ELSE '4. Hardcore (200hrs+)'
    END AS player_tier,
    COUNT(*) AS total_reviews,
    SUM(voted_up) AS positive,
    COUNT(*) - SUM(voted_up) AS negative,
    ROUND(SUM(voted_up) * 100.0 / COUNT(*), 2) AS positive_rate
FROM gta_clean
GROUP BY player_tier
ORDER BY player_tier;
```

**Finding:** U-shaped curve — Moderate (92.92%) and Dedicated (92.30%) above baseline. Casual (84.91%) and Hardcore (86.92%) both below. Most invested and least invested players are the least satisfied.

---

### EDA Query 5 — Churn vs Frustration (Saved as View)

```sql
CREATE VIEW vw_churn_vs_frustration AS
SELECT 
    CASE 
        WHEN author_playtime_last_two_weeks > 0 THEN 'Frustrated but still playing'
        ELSE 'Churned - no recent activity'
    END AS player_status,
    COUNT(*) AS total_negative_reviews,
    ROUND(COUNT(*) * 100.0 / SUM(COUNT(*)) OVER(), 2) AS percentage
FROM gta_clean
WHERE voted_up = 0
GROUP BY player_status;
```

**Finding:** 70.57% of negative reviewers have already stopped playing. This is a churn crisis not a frustration crisis. The intervention window is closing.

---

### EDA Query 6 — First Time vs Habitual Reviewers (Saved as View)

```sql
CREATE VIEW vw_sentiment_by_reviewer_type AS
SELECT 
    CASE 
        WHEN author_num_reviews = 1 THEN 'First Time Reviewer'
        WHEN author_num_reviews BETWEEN 2 AND 5 THEN 'Occasional Reviewer'
        ELSE 'Habitual Reviewer (6+)'
    END AS reviewer_type,
    COUNT(*) AS total_reviews,
    SUM(voted_up) AS positive,
    COUNT(*) - SUM(voted_up) AS negative,
    ROUND(SUM(voted_up) * 100.0 / COUNT(*), 2) AS positive_rate
FROM gta_clean
GROUP BY reviewer_type
ORDER BY reviewer_type;
```

**Finding:** Habitual reviewers at 85.76% — nearly 5 points below baseline. The most experienced and credible voices on Steam are the most negative.

---

### EDA Query 7 — Purchase and Loyalty Segments (Saved as View)

```sql
CREATE VIEW vw_sentiment_by_segment AS
SELECT 
    CASE 
        WHEN steam_purchase = 1 AND recieved_for_free = 0 THEN 'Paid Player'
        WHEN steam_purchase = 0 AND recieved_for_free = 1 THEN 'Free Player'
        ELSE 'Other'
    END AS player_segment,
    COUNT(*) AS total_reviews,
    SUM(voted_up) AS positive,
    COUNT(*) - SUM(voted_up) AS negative,
    ROUND(SUM(voted_up) * 100.0 / COUNT(*), 2) AS positive_rate
FROM gta_clean
GROUP BY player_segment
ORDER BY positive_rate ASC;
```

**Finding:** Sentiment nearly identical across paid, free and other segments — suggesting the product problem affects all player types equally regardless of spend.

---

### EDA Query 8 — Viral Complaint Detection (Saved as View)

```sql
CREATE VIEW vw_viral_complaints AS
SELECT 
    id,
    LEFT(review, 100) AS review_preview,
    votes_up,
    comment_count,
    author_playtime_forever,
    (votes_up + comment_count) AS amplification_score
FROM gta_clean
WHERE voted_up = 0
    AND (votes_up + comment_count) > 0
ORDER BY amplification_score DESC
LIMIT 15;
```

**Finding:** Top amplified complaint scored 530 upvotes from a player with 857 hours. Every single top 15 complaint references GTA Online specifically — hackers, account issues, monetisation, server problems. The base game is not the problem.

---

### EDA Query 9 — Early Access Column Validation

```sql
SELECT 
    written_during_early_access,
    COUNT(*) AS total
FROM gta_clean
GROUP BY written_during_early_access;
```

**Finding:** All 52,099 rows returned 0 — GTA Online had no early access period on Steam. Column noted as not applicable and documented accordingly.

---

### EDA Query 10 — Executive Summary View

```sql
CREATE VIEW vw_executive_summary AS
SELECT
    (SELECT COUNT(*) FROM gta_clean) AS total_reviews,
    (SELECT ROUND(SUM(voted_up) * 100.0 / COUNT(*), 2) FROM gta_clean) AS overall_positive_rate,
    (SELECT COUNT(*) FROM gta_clean WHERE voted_up = 0) AS total_negative_reviews,
    (SELECT ROUND(COUNT(*) * 100.0 / (SELECT COUNT(*) FROM gta_clean WHERE voted_up = 0), 2)
        FROM gta_clean WHERE voted_up = 0 AND author_playtime_last_two_weeks = 0) AS churned_pct,
    (SELECT ROUND(COUNT(*) * 100.0 / (SELECT COUNT(*) FROM gta_clean WHERE voted_up = 0), 2)
        FROM gta_clean WHERE voted_up = 0 AND author_playtime_last_two_weeks > 0) AS frustrated_still_playing_pct,
    (SELECT ROUND(SUM(voted_up) * 100.0 / COUNT(*), 2)
        FROM gta_clean WHERE author_playtime_forever >= 12000) AS hardcore_positive_rate,
    (SELECT ROUND(SUM(voted_up) * 100.0 / COUNT(*), 2)
        FROM gta_clean WHERE author_num_reviews >= 6) AS habitual_reviewer_positive_rate,
    (SELECT MAX(votes_up + comment_count)
        FROM gta_clean WHERE voted_up = 0) AS highest_amplification_score;
```

**Output:**

| Metric | Value |
|---|---|
| Total Reviews | 52,099 |
| Overall Positive Rate | 90.71% |
| Total Negative Reviews | 4,839 |
| Churned % | 70.57% |
| Frustrated Still Playing % | 29.43% |
| Hardcore Positive Rate | 86.92% |
| Habitual Reviewer Positive Rate | 85.76% |
| Highest Amplification Score | 530 |

---

## Part 2 — Python

### What and Why

Python was the intelligence layer of the project — it did what SQL fundamentally cannot. SQL told us 90.71% of players gave a thumbs up. Python revealed that only 56% of reviews contained genuinely positive language, meaning 17,000 players clicked thumbs up but wrote neutral or lukewarm text. That gap between the surface metric and the written reality is a finding SQL could never have surfaced. VADER sentiment scoring gave every review a score between -1 and +1, NLP word frequency analysis read all 4,839 negative reviews and extracted the dominant complaint themes, and a composite amplification risk score was engineered from votes, comments, playtime and reviewer credibility. Nine new columns were added to the dataset and exported as an enriched CSV that feeds directly into Power BI.

---

### Setup

```python
# Install
!pip install vaderSentiment --quiet
!pip install wordcloud --quiet

# Import
import pandas as pd
import numpy as np
import matplotlib.pyplot as plt
import seaborn as sns
from vaderSentiment.vaderSentiment import SentimentIntensityAnalyzer
from wordcloud import WordCloud
import nltk
from nltk.corpus import stopwords
from nltk.tokenize import word_tokenize
from collections import Counter
import warnings
warnings.filterwarnings('ignore')

nltk.download('stopwords', quiet=True)
nltk.download('punkt', quiet=True)
nltk.download('punkt_tab', quiet=True)
```

---

### Data Loading and Cleaning

```python
# Load
df = pd.read_csv('/content/GTADATA.csv', sep='|', encoding='utf-8', low_memory=False)

# Convert datetimes
df['created'] = pd.to_datetime(df['created'], dayfirst=True)
df['author_last_played'] = pd.to_datetime(df['author_last_played'], dayfirst=True)

# Convert playtime from minutes to hours
df['playtime_forever_hrs'] = round(df['author_playtime_forever'] / 60, 1)
df['playtime_at_review_hrs'] = round(df['author_playtime_at_review'] / 60, 1)
df['playtime_last_2weeks_hrs'] = round(df['author_playtime_last_two_weeks'] / 60, 1)
```

---

### VADER Sentiment Scoring

```python
analyzer = SentimentIntensityAnalyzer()

def get_sentiment_score(text):
    if pd.isna(text) or text == '':
        return 0
    return analyzer.polarity_scores(str(text))['compound']

df['sentiment_score'] = df['review'].apply(get_sentiment_score)
```

**Finding:**

| Method | Positive | Neutral | Negative |
|---|---|---|---|
| SQL voted_up | 47,260 (90.71%) | — | 4,839 (9.29%) |
| VADER | 29,170 (56%) | 16,984 (32.6%) | 5,945 (11.4%) |

17,000 players clicked thumbs up but wrote neutral text. Silent dissatisfaction hidden inside the headline metric.

---

### Player Tier Segmentation

```python
def assign_tier(hrs):
    if hrs < 10:
        return '1. Casual\n(0-10hrs)'
    elif hrs < 50:
        return '2. Moderate\n(10-50hrs)'
    elif hrs < 200:
        return '3. Dedicated\n(50-200hrs)'
    else:
        return '4. Hardcore\n(200hrs+)'

df['player_tier'] = df['playtime_forever_hrs'].apply(assign_tier)
```

---

### NLP — Negative Review Word Frequency

```python
negative_reviews = df[df['voted_up'] == 0]['review'].dropna()

stop_words = set(stopwords.words('english'))
custom_stopwords = {'game', 'gta', 'rockstar', 'play', 'get', 'one', 
                    'like', 'also', 'would', 'still', 'even', 'good',
                    'great', 'really', 'much', 'go', 'got', 'make',
                    'time', 'want', 'know', 'way', 'thing', 'nan'}
stop_words.update(custom_stopwords)

all_words = []
for review in negative_reviews:
    tokens = word_tokenize(str(review).lower())
    words = [w for w in tokens if w.isalpha() and w not in stop_words and len(w) > 2]
    all_words.extend(words)

word_freq = Counter(all_words)
```

**Top 10 complaint words:**

| Word | Frequency | Business Meaning |
|---|---|---|
| online | 1,264 | Live service is the core problem |
| account | 641 | Account management failures |
| money | 554 | Monetisation backlash |
| games | 400 | Comparative dissatisfaction |
| buy | 395 | Active word of mouth damage |
| hackers | 387 | Security and integrity failure |
| bad | 381 | General quality decline |
| steam | 364 | Platform friction issues |
| modders | 349 | Cheating environment |
| fun | 348 | Core product failing at its most basic job |

---

### Amplification Risk Score

```python
df['amplification_risk'] = (
    df['votes_up'] * 2 +
    df['comment_count'] * 1.5 +
    (df['playtime_forever_hrs'] / df['playtime_forever_hrs'].max()) * 10 +
    df['author_num_reviews'].apply(lambda x: 5 if x >= 6 else 0)
)
```

**Weighting logic:**
- `votes_up × 2` — community agreement is the strongest amplification signal
- `comment_count × 1.5` — engagement depth shows the complaint sparked conversation
- `playtime proportion × 10` — high investment player complaints carry more credibility
- `+5 for habitual reviewers` — experienced reviewers have larger audiences and more trust

---

### Churn and Segment Classification

```python
def get_segment(row):
    if row['steam_purchase'] == 1 and row['recieved_for_free'] == 0:
        return 'Paid Player'
    elif row['recieved_for_free'] == 1:
        return 'Free Player'
    else:
        return 'Other'

df['player_segment'] = df.apply(get_segment, axis=1)

df['churn_status'] = df.apply(
    lambda row: 'Churned' if row['voted_up'] == 0 and row['playtime_last_2weeks_hrs'] == 0
    else ('Frustrated' if row['voted_up'] == 0 and row['playtime_last_2weeks_hrs'] > 0
    else 'Active'), axis=1
)
```

---

### Charts Produced

| Chart | File | Business Purpose |
|---|---|---|
| Sentiment Distribution | chart1_sentiment_distribution.png | Reveals 17,000 neutral reviews hiding in positive rate |
| Sentiment by Player Tier | chart2_sentiment_by_tier.png | Confirms U-shaped loyalty curve |
| Negative Word Frequency | chart3_negative_words.png | Identifies top complaint themes |
| Negative Word Cloud | chart4_negative_wordcloud.png | Executive visual of complaint landscape |
| Positive Word Cloud | chart5_positive_wordcloud.png | Contrast — what players actually love |
| Correlation Heatmap | chart6_correlation_heatmap.png | Statistical relationships between all variables |
| Churn vs Frustration | chart7_churn_vs_frustration.png | Quantifies the crisis vs recoverable split |
| Amplification Risk | chart8_amplification_risk.png | Top 15 highest risk negative reviews |
| Reviewer Type Sentiment | chart9_reviewer_type_sentiment.png | Habitual reviewer credibility gap |
| Sentiment Trend | chart10_sentiment_trend.png | Timeline of when decline started |

---

### Export

```python
df.to_csv('/content/GTA_FINAL_ENRICHED.csv', index=False)
```

Final dataset: 52,099 rows × 26 columns

---

## Part 3 — Excel

### What and Why

Excel was where data became a commercial decision tool. SQL and Python produced findings. Excel attached pound values to them, built a risk matrix any manager could read in ten seconds, and produced an executive summary table that translates every analytical finding into a specific business action. Three sheets were built on top of the enriched CSV using live formulas — meaning if the underlying data updates, every calculation updates automatically. The business problem Excel solved that no other tool could: it answered the question every boardroom eventually asks — what is this actually costing us?

---

### Sheet 1 — Raw Data
Full GTA_FINAL_ENRICHED.csv imported as the single source of truth. All other sheets pull from this using live cell references.

---

### Sheet 2 — Retention Risk Matrix

**Pivot Table Setup:**
- Rows: `player_tier`
- Columns: `churn_status`
- Values: Count of `id`

**Churn Rate Formula (column H):**
```excel
=ROUND((D5/G5)*100,1)
```
Churned count divided by Grand Total × 100

**Average Sentiment Formula (column I):**
```excel
=ROUND(AVERAGEIF('Raw Data'!U:U,'Retention Risk Matrix'!B5,'Raw Data'!T:T),3)
```
Pulls average sentiment score for each player tier directly from Raw Data

**Output:**

| Player Tier | Churn Rate % | Avg Sentiment | Status |
|---|---|---|---|
| Casual (0-10hrs) | 13.9% | 0.206 | 🔴 Highest Risk |
| Moderate (10-50hrs) | 5.6% | 0.260 | 🟢 Healthy |
| Dedicated (50-200hrs) | 5.5% | 0.260 | 🟢 Healthy |
| Hardcore (200hrs+) | 8.2% | 0.234 | 🟡 Watch Closely |

---

### Sheet 3 — Revenue Impact Model

**Formulas:**

```excel
C4: =COUNTA('Raw Data'!A:A)-1                          -- Total reviews
C5: =COUNTIF('Raw Data'!Y:Y,"Paid Player")             -- Total paid players
C6: =COUNTIFS('Raw Data'!Y:Y,"Paid Player",'Raw Data'!Z:Z,"Churned")    -- Churned paid
C7: =COUNTIFS('Raw Data'!Y:Y,"Paid Player",'Raw Data'!Z:Z,"Frustrated") -- At risk paid
C8: =ROUND(C6/C5*100,1)&"%"                            -- Churn rate
C9: 30                                                  -- Assumption cell (£ per player)
C10: =C6*C9                                            -- Revenue already lost
C11: =C7*C9                                            -- Revenue at risk
```

**Output:**

| Metric | Value |
|---|---|
| Total Paid Players | 45,277 |
| Churned Paid Players | 3,019 |
| Frustrated Paid Players | 1,213 |
| Churn Rate | 6.7% |
| Revenue Already Lost | £90,570 |
| Revenue At Risk | £36,390 |
| **Total Exposure** | **£126,960** |

Note: £30 is a conservative base price estimate. Real exposure is higher when in-game purchases and DLC are included.

---

### Sheet 4 — Executive Summary

| # | Business Question | Key Finding | Data Evidence | Business Implication | Recommended Action |
|---|---|---|---|---|---|
| 1 | Is sentiment declining and when did it start? | May 2024 was lowest sentiment month at 0.11 — 56% below overall mean | 52,099 reviews, VADER scoring, monthly trend | Sentiment never fully recovered — structural problem not temporary | Investigate May 2024 patch notes and pricing changes |
| 2 | Are highest invested players becoming loudest critics? | Hardcore players score 0.234 vs 0.26 for moderate players | Playtime tier segmentation across 12,671 hardcore reviewers | Most invested players below average satisfaction — loyalty erosion | Prioritise hardcore player feedback in next development cycle |
| 3 | Is this a churn crisis or frustration crisis? | 72% of negative reviewers have zero recent playtime | playtime_last_two_weeks analysis across 4,839 negative reviews | Churn crisis — win-back campaigns needed immediately | Launch targeted win-back campaign for churned paid players within 30 days |
| 4 | Are experienced reviewers driving negative sentiment? | Habitual reviewers score 0.187 — 25% below mean | reviewer_type segmentation across 13,395 habitual reviewers | Most credible Steam voices are most negative | Address top 5 viral complaints directly in next community update |
| 5 | What are players actually complaining about? | Online, account, money, hackers dominate — 1,264 mentions of online | NLP analysis of 4,839 negative reviews | GTA Online infrastructure is the problem — not the base game | Dedicate next patch entirely to anti-cheat, account security and server stability |

---

## Part 4 — Power BI

### What and Why

Power BI was the final and most visible layer of the project — the point where every finding became interactive and accessible to a non-technical business audience. SQL, Python and Excel produced static outputs. Power BI made everything live. A Head of Product can open this report, click Hardcore Players on any slicer and every chart, card and table across every page updates instantly to show only that segment. Fifteen DAX measures were written to power the calculations, five pages were built each answering a distinct business question, and the final page reframes the entire project from post-mortem to recovery strategy by identifying the 1,213 players still reachable and the £36,390 still recoverable.

---

### Data Connection
Connected directly to `GTA_FINAL_ENRICHED.csv` via Get Data → Text/CSV in Power BI Desktop.

---

### DAX Measures

```dax
Total Reviews = 
    COUNTROWS(GTA_FINAL_ENRICHED)

Positive Rate % = 
    DIVIDE(
        COUNTROWS(FILTER(GTA_FINAL_ENRICHED, GTA_FINAL_ENRICHED[voted_up] = 1)),
        COUNTROWS(GTA_FINAL_ENRICHED)
    ) * 100

Avg Sentiment Score = 
    AVERAGE(GTA_FINAL_ENRICHED[sentiment_score])

Total Churned Players = 
    COUNTROWS(FILTER(GTA_FINAL_ENRICHED, GTA_FINAL_ENRICHED[churn_status] = "Churned"))

Churn Rate % = 
    DIVIDE([Total Churned Players], [Total Reviews]) * 100

Frustrated Players = 
    COUNTROWS(FILTER(GTA_FINAL_ENRICHED, GTA_FINAL_ENRICHED[churn_status] = "Frustrated"))

Total Negative Reviews = 
    COUNTROWS(FILTER(GTA_FINAL_ENRICHED, GTA_FINAL_ENRICHED[voted_up] = 0))

Revenue Already Lost = 
    COUNTROWS(
        FILTER(GTA_FINAL_ENRICHED, 
        GTA_FINAL_ENRICHED[churn_status] = "Churned" && 
        GTA_FINAL_ENRICHED[player_segment] = "Paid Player")
    ) * 30

Revenue At Risk = 
    COUNTROWS(
        FILTER(GTA_FINAL_ENRICHED, 
        GTA_FINAL_ENRICHED[churn_status] = "Frustrated" && 
        GTA_FINAL_ENRICHED[player_segment] = "Paid Player")
    ) * 30

Avg Sentiment Churned = 
    CALCULATE(
        AVERAGE(GTA_FINAL_ENRICHED[sentiment_score]), 
        GTA_FINAL_ENRICHED[churn_status] = "Churned"
    )

Avg Sentiment Paid Players = 
    CALCULATE(
        AVERAGE(GTA_FINAL_ENRICHED[sentiment_score]), 
        GTA_FINAL_ENRICHED[player_segment] = "Paid Player"
    )

Hardcore Player Churn Rate % = 
    DIVIDE(
        COUNTROWS(FILTER(GTA_FINAL_ENRICHED, 
            GTA_FINAL_ENRICHED[player_tier] = "4. Hardcore\n(200hrs+)" && 
            GTA_FINAL_ENRICHED[churn_status] = "Churned")),
        COUNTROWS(FILTER(GTA_FINAL_ENRICHED, 
            GTA_FINAL_ENRICHED[player_tier] = "4. Hardcore\n(200hrs+)"))
    ) * 100

Dynamic Title = 
    "GTA Online Player Intelligence Report — " & FORMAT(TODAY(), "MMMM YYYY")

Total Revenue Exposure = 
    [Revenue Already Lost] + [Revenue At Risk]

Recovery Value = 
    COUNTROWS(
        FILTER(GTA_FINAL_ENRICHED, 
        GTA_FINAL_ENRICHED[churn_status] = "Frustrated" && 
        GTA_FINAL_ENRICHED[player_segment] = "Paid Player")
    ) * 30
```

---

### Report Pages

**Page 1 — Executive Overview**
5 KPI cards: Total Reviews, Positive Rate %, Churn Rate %, Total Revenue Exposure, Avg Sentiment Score. Sentiment trend line over time. Page navigator for seamless movement. Canvas background #0D1B2A.

**Page 2 — Player Sentiment Analysis**
Slicer: player_segment. Bar chart: sentiment by player tier. Bar chart: sentiment by reviewer type. Bar chart: sentiment by purchase type. Scatter plot: playtime vs sentiment. Donut chart: reviewer type distribution.

**Page 3 — Churn & Retention Risk**
Slicer: player_tier. Bar chart: churned vs frustrated count. Bar chart: churn rate by tier. KPI visual: churned vs at risk. Stacked bar: churn status by segment. Table: retention risk summary with churn rate, sentiment and tier combined.

**Page 4 — Voice of the Player**
Slicer: churn_status. Bar chart: amplification risk by churn status and tier. Table: risk summary combining sentiment, churn rate and amplification risk. Combo chart: review volume vs sentiment over time.

**Page 5 — Recovery Roadmap**
KPI card: Revenue Already Lost (£90,570) — red. KPI card: Recovery Value (£36,390) — orange. Bar chart: profile of frustrated players still reachable by tier. Scatter plot: intervention window showing playtime vs sentiment for frustrated players only.

---

## Key Findings Summary

| Finding | Metric | Business Implication |
|---|---|---|
| Overall sentiment baseline | 90.71% positive | Reference point for all segment comparisons |
| VADER vs voted_up gap | 56% vs 90.71% positive | 17,000 players are silently dissatisfied |
| Churn crisis confirmed | 72% of negative reviewers already gone | Win-back window is closing |
| Hardcore player loyalty erosion | 8.2% churn, 0.234 sentiment | Most invested players deteriorating fastest |
| Habitual reviewer credibility gap | 0.187 sentiment — 25% below mean | Trusted voices are most negative |
| Top complaint — online infrastructure | 1,264 mentions of online | Live service not base game is the problem |
| Revenue already lost | £90,570 | 3,019 churned paid players × £30 |
| Revenue still recoverable | £36,390 | 1,213 frustrated paid players still active |
| Total revenue exposure | £126,960 | Conservative estimate excluding in-game spend |

---

## Repository Structure

```
GTA-Player-Intelligence-Report/
│
├── README.md
├── data/
│   └── data_dictionary.md
├── sql/
│   ├── 01_create_table.sql
│   ├── 02_create_clean_view.sql
│   ├── 03_eda_queries.sql
│   └── 04_saved_views.sql
├── python/
│   ├── 01_data_cleaning_export.ipynb
│   ├── 02_sentiment_scoring.ipynb
│   ├── 03_nlp_analysis.ipynb
│   └── 04_visualisations.ipynb
├── excel/
│   └── GTA_Player_Intelligence.xlsx
├── powerbi/
│   └── GTA_Player_Intelligence.pbix
└── outputs/
    ├── charts/
    │   ├── chart1_sentiment_distribution.png
    │   ├── chart2_sentiment_by_tier.png
    │   ├── chart3_negative_words.png
    │   ├── chart4_negative_wordcloud.png
    │   ├── chart5_positive_wordcloud.png
    │   ├── chart6_correlation_heatmap.png
    │   ├── chart7_churn_vs_frustration.png
    │   ├── chart8_amplification_risk.png
    │   ├── chart9_reviewer_type_sentiment.png
    │   └── chart10_sentiment_trend.png
    └── executive_summary.pdf
```

---

## Skills Demonstrated

| Skill | Tool | How |
|---|---|---|
| SQL | MySQL | 10 EDA queries, 7 views, window functions, CASE statements, STR_TO_DATE, subqueries |
| Python | Google Colab | VADER NLP, word frequency analysis, correlation heatmap, 10 charts, feature engineering |
| Data Cleaning | Python + SQL | Encoding fix, boolean conversion, datetime parsing, null handling, column engineering |
| Statistical Analysis | Python | Correlation matrix across 12 variables, sentiment distribution analysis |
| Business Modelling | Excel | Revenue impact calculator, retention risk matrix, assumption-driven financial model |
| DAX | Power BI | 15 custom measures using COUNTROWS, FILTER, DIVIDE, CALCULATE, AVERAGE, FORMAT |
| Data Visualisation | Power BI + Python | 20+ Power BI visuals, 10 Python charts, word clouds, heatmaps |
| Data Storytelling | Power BI | 5-page narrative structure moving from diagnosis to recovery strategy |
| Customer Segmentation | All tools | Player tier, reviewer type, purchase segment, churn status across all analysis |
| Root Cause Analysis | NLP + SQL | Identified GTA Online infrastructure as the specific source of sentiment decline |

---

## How to Run

**SQL:**
1. Install MySQL 8.0+
2. Run scripts in order: 01 → 02 → 03 → 04
3. Note: column `recieved_for_free` uses original CSV spelling throughout

**Python:**
1. Open Google Colab
2. Upload `GTADATA.csv` (pipe delimited, utf-8)
3. Run notebooks in order: 01 → 02 → 03 → 04
4. Download `GTA_FINAL_ENRICHED.csv` from final cell

**Excel:**
1. Open `GTA_Player_Intelligence.xlsx`
2. If prompted to update links point to your local `GTA_FINAL_ENRICHED.csv`

**Power BI:**
1. Open `GTA_Player_Intelligence.pbix` in Power BI Desktop
2. Go to Transform Data → update file path to your local `GTA_FINAL_ENRICHED.csv`
3. Click Close and Apply

---

*Project by [Your Name] | [LinkedIn] | [Portfolio]*
