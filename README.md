# Project: Lyrics and Persistence in Spotify Top-50

## 1. Objective

The goal of this project is to study whether **lyrical content is associated with the persistence of songs in the Spotify Top-50 chart in the United States**.

Persistence is defined as the **number of days a song remains in the Top-50 ranking**.

The project integrates:

- Spotify chart data
- Song lyrics (text data)
- Text mining techniques
- Statistical analysis

The central idea is to examine whether characteristics of lyrics (sentiment, themes, complexity, etc.) help explain why some songs stay longer in the charts.

---

## 2. Data Sources

### Spotify Charts Dataset

The base dataset is sourced from Kaggle (`asaniczka/top-spotify-songs-in-73-countries-daily-updated`) and contains daily rankings of songs in Spotify charts across 73 countries.

Relevant variables include:

- `spotify_id` — unique Spotify track identifier
- `name` — song title
- `artists` — one or more artists (comma-separated)
- `daily_rank` — chart position on a given day
- `snapshot_date` — date of the chart snapshot
- `country` — two-letter country code
- `popularity` — Spotify popularity score
- Audio features: `danceability`, `energy`, `valence`, `tempo`, `loudness`, `speechiness`, `acousticness`, `instrumentalness`

### Lyrics Data

Lyrics are retrieved from the **Genius database** using the Python library `lyricsgenius`. The Genius API returns song metadata, result rankings, and access to full lyrics pages.

---

## 3. Dataset Preparation

### Step 1 — Filter to United States

Only rows where `country == 'US'` are kept. This restricts the analysis to a single music market.

### Step 2 — Compute Song Persistence

For each `spotify_id`, persistence is defined as the number of **unique `snapshot_date` values** in the US chart data:

```python
days_in_top50 = df_us.groupby('spotify_id')['snapshot_date'].nunique()
```

### Step 3 — Build Song-Level Table

Duplicate rows are dropped (keeping the earliest appearance per song), resulting in one row per unique song with the following fields:

- `spotify_id`
- `name`
- `artists`
- `album_name`
- `album_release_date`
- `days_in_top50`

This yielded **1,060 unique songs** in the US Top-50.

---

## 4. Lyrics Collection

Lyrics are retrieved using the Genius API via the `lyricsgenius` library. The client is configured as follows:

```python
genius = lyricsgenius.Genius(
    GENIUS_TOKEN,
    skip_non_songs=True,
    timeout=10,
    retries=1        # prevents hanging on failed requests
)
genius.verbose = False
```

### Search Query Cleaning (`clean_for_search`)

Spotify song titles frequently include `(feat. Artist)` suffixes, and the `artists` field often contains multiple comma-separated names. Feeding the raw strings directly into the Genius search produces poor results (e.g., unrelated pages like fan compilations or transcription documents).

To fix this:
- `(feat. ...)` is stripped from the title using a regex
- Only the **first artist** (before the first comma) is used in the query

```python
def clean_for_search(title, artist):
    title_clean = re.sub(r'\(feat\.?[^)]*\)', '', title, flags=re.IGNORECASE).strip()
    artist_clean = artist.split(',')[0].strip()
    return title_clean, artist_clean
```

### Result Validation (`title_matches`)

The Genius API sometimes returns completely unrelated results as the top hit. To catch this, each candidate result's title is compared against the searched title using **word overlap**:

```python
def title_matches(query_title, result_title, threshold=0.5):
    query_words = set(re.sub(r'[^\w\s]', '', query_title.lower()).split())
    result_words = set(re.sub(r'[^\w\s]', '', result_title.lower()).split())
    if not query_words:
        return False
    return len(query_words & result_words) / len(query_words) >= threshold
```

If fewer than 50% of the query title's words appear in the result title, the hit is skipped.

### Translation Filtering (`is_translated_title`)

Genius hosts translated versions of lyrics (e.g., *"Blinding Lights (Tradução em Português)"*). These are detected and skipped by checking for known translation keywords in the result title:

```python
def is_translated_title(title):
    patterns = [r'\(tradução\)', r'\(traducción\)', r'\(translation\)', r'\(letra\)', r'\(lyrics\)']
    return any(re.search(p, title, re.IGNORECASE) for p in patterns)
```

### Retrieval Loop

For each song, the top 5 Genius search hits are examined in order. Each hit is:
1. Checked for translation keywords → skipped if found
2. Checked for title similarity → skipped if overlap < 50%
3. Fetched by `song_id` to retrieve the full lyrics text

The first hit that passes all checks is accepted. If none pass, the song is logged as a failure.

A `time.sleep(0.5)` delay is applied between requests to avoid rate-limiting.

### Sanity Check (10% Random Sample)

After the full run, a random 10% sample of songs (fixed with `random_state=42` for reproducibility) is re-run with `verbose=True`, printing every intermediate step (query sent, hits returned, which hits were skipped and why, final result). This serves as a manual quality check on the retrieval logic.

---


# 5. Text Cleaning

Lyrics contain structural elements such as:

<pre class="overflow-visible! px-0!" data-start="3278" data-end="3313"><div class="relative w-full mt-4 mb-1"><div class=""><div class="relative"><div class="h-full min-h-0 min-w-0"><div class="h-full min-h-0 min-w-0"><div class="border border-token-border-light border-radius-3xl corner-superellipse/1.1 rounded-3xl"><div class="h-full w-full border-radius-3xl bg-token-bg-elevated-secondary corner-superellipse/1.1 overflow-clip rounded-3xl lxnfua_clipPathFallback"><div class="pointer-events-none absolute end-1.5 top-1 z-2 md:end-2 md:top-1"></div><div class="pe-11 pt-3"><div class="relative z-0 flex max-w-full"><div id="code-block-viewer" dir="ltr" class="q9tKkq_viewer cm-editor z-10 light:cm-light dark:cm-light flex h-full w-full flex-col items-stretch ͼk ͼy"><div class="cm-scroller"><div class="cm-content q9tKkq_readonly"><span>[Verse 1]</span><br/><span>[Chorus]</span><br/><span>[Bridge]</span></div></div></div></div></div></div></div></div></div><div class=""><div class=""></div></div></div></div></div></pre>

Cleaning steps include:

* removing section headers
* converting text to lowercase
* removing punctuation
* removing extra whitespace.

The goal is to obtain a clean text corpus suitable for analysis.

---

# 6. Language Filtering

Some Genius pages correspond to** ** **translated lyrics** .

To ensure dataset consistency, language detection is applied using the library:

<pre class="overflow-visible! px-0!" data-start="3684" data-end="3702"><div class="relative w-full mt-4 mb-1"><div class=""><div class="relative"><div class="h-full min-h-0 min-w-0"><div class="h-full min-h-0 min-w-0"><div class="border border-token-border-light border-radius-3xl corner-superellipse/1.1 rounded-3xl"><div class="h-full w-full border-radius-3xl bg-token-bg-elevated-secondary corner-superellipse/1.1 overflow-clip rounded-3xl lxnfua_clipPathFallback"><div class="pointer-events-none absolute end-1.5 top-1 z-2 md:end-2 md:top-1"></div><div class="pe-11 pt-3"><div class="relative z-0 flex max-w-full"><div id="code-block-viewer" dir="ltr" class="q9tKkq_viewer cm-editor z-10 light:cm-light dark:cm-light flex h-full w-full flex-col items-stretch ͼk ͼy"><div class="cm-scroller"><div class="cm-content q9tKkq_readonly"><span>langdetect</span></div></div></div></div></div></div></div></div></div><div class=""><div class=""></div></div></div></div></div></pre>

Only songs with detected language:

<pre class="overflow-visible! px-0!" data-start="3740" data-end="3755"><div class="relative w-full mt-4 mb-1"><div class=""><div class="relative"><div class="h-full min-h-0 min-w-0"><div class="h-full min-h-0 min-w-0"><div class="border border-token-border-light border-radius-3xl corner-superellipse/1.1 rounded-3xl"><div class="h-full w-full border-radius-3xl bg-token-bg-elevated-secondary corner-superellipse/1.1 overflow-clip rounded-3xl lxnfua_clipPathFallback"><div class="pointer-events-none absolute end-1.5 top-1 z-2 md:end-2 md:top-1"></div><div class="pe-11 pt-3"><div class="relative z-0 flex max-w-full"><div id="code-block-viewer" dir="ltr" class="q9tKkq_viewer cm-editor z-10 light:cm-light dark:cm-light flex h-full w-full flex-col items-stretch ͼk ͼy"><div class="cm-scroller"><div class="cm-content q9tKkq_readonly"><span>English</span></div></div></div></div></div></div></div></div></div><div class=""><div class=""></div></div></div></div></div></pre>

are retained.

This ensures the corpus consists of** ** **texts written in a single language** , which is important for most text mining methods.

---

# 7. Text Feature Generation

Once lyrics are cleaned, textual features can be constructed.

## Basic Text Features

Examples include:

* word count
* vocabulary size
* lexical diversity
* average word length.

---

## Sentiment Features

Lyrics can be analyzed using sentiment dictionaries to compute:

* frequency of positive words
* frequency of negative words
* overall sentiment score.

---

## Topic Features

Topic modeling methods such as** ****Latent Dirichlet Allocation (LDA)** can be used to identify common themes in lyrics.

Examples of possible topics:

* love and relationships
* nightlife
* sadness
* wealth and status.

---

## Lexical Categories

Custom dictionaries may capture specific lyrical themes, such as:

* love words
* sadness words
* profanity
* violence
* party or nightlife terms.

---

## Vector Representations

Lyrics can also be converted into numerical vectors using:

* Bag-of-Words
* TF-IDF representations
* embeddings.

These representations enable machine learning models.

---

# 8. Final Dataset Structure

The final dataset contains one row per song and includes:

<pre class="overflow-visible! px-0!" data-start="5009" data-end="5132"><div class="relative w-full mt-4 mb-1"><div class=""><div class="relative"><div class="h-full min-h-0 min-w-0"><div class="h-full min-h-0 min-w-0"><div class="border border-token-border-light border-radius-3xl corner-superellipse/1.1 rounded-3xl"><div class="h-full w-full border-radius-3xl bg-token-bg-elevated-secondary corner-superellipse/1.1 overflow-clip rounded-3xl lxnfua_clipPathFallback"><div class="pointer-events-none absolute end-1.5 top-1 z-2 md:end-2 md:top-1"></div><div class="pe-11 pt-3"><div class="relative z-0 flex max-w-full"><div id="code-block-viewer" dir="ltr" class="q9tKkq_viewer cm-editor z-10 light:cm-light dark:cm-light flex h-full w-full flex-col items-stretch ͼk ͼy"><div class="cm-scroller"><div class="cm-content q9tKkq_readonly"><span>spotify_id</span><br/><span>name</span><br/><span>artist</span><br/><span>album_name</span><br/><span>album_release_date</span><br/><span>days_in_top50</span><br/><span>lyrics</span><br/><span>clean_lyrics</span><br/><span>text_features</span><br/><span>audio_features</span></div></div></div></div></div></div></div></div></div><div class=""><div class=""></div></div></div></div></div></pre>

This structure combines:

* textual information
* musical features
* performance outcomes.

---

# 9. Analysis Strategy

The main outcome variable is:

<pre class="overflow-visible! px-0!" data-start="5286" data-end="5307"><div class="relative w-full mt-4 mb-1"><div class=""><div class="relative"><div class="h-full min-h-0 min-w-0"><div class="h-full min-h-0 min-w-0"><div class="border border-token-border-light border-radius-3xl corner-superellipse/1.1 rounded-3xl"><div class="h-full w-full border-radius-3xl bg-token-bg-elevated-secondary corner-superellipse/1.1 overflow-clip rounded-3xl lxnfua_clipPathFallback"><div class="pointer-events-none absolute end-1.5 top-1 z-2 md:end-2 md:top-1"></div><div class="pe-11 pt-3"><div class="relative z-0 flex max-w-full"><div id="code-block-viewer" dir="ltr" class="q9tKkq_viewer cm-editor z-10 light:cm-light dark:cm-light flex h-full w-full flex-col items-stretch ͼk ͼy"><div class="cm-scroller"><div class="cm-content q9tKkq_readonly"><span>days_in_top50</span></div></div></div></div></div></div></div></div></div><div class=""><div class=""></div></div></div></div></div></pre>

Possible analyses include:

### Regression Analysis

Example model:

<pre class="overflow-visible! px-0!" data-start="5378" data-end="5436"><div class="relative w-full mt-4 mb-1"><div class=""><div class="relative"><div class="h-full min-h-0 min-w-0"><div class="h-full min-h-0 min-w-0"><div class="border border-token-border-light border-radius-3xl corner-superellipse/1.1 rounded-3xl"><div class="h-full w-full border-radius-3xl bg-token-bg-elevated-secondary corner-superellipse/1.1 overflow-clip rounded-3xl lxnfua_clipPathFallback"><div class="pointer-events-none absolute end-1.5 top-1 z-2 md:end-2 md:top-1"></div><div class="pe-11 pt-3"><div class="relative z-0 flex max-w-full"><div id="code-block-viewer" dir="ltr" class="q9tKkq_viewer cm-editor z-10 light:cm-light dark:cm-light flex h-full w-full flex-col items-stretch ͼk ͼy"><div class="cm-scroller"><div class="cm-content q9tKkq_readonly"><span>days_in_top50 = f(lyrics_features, audio_features)</span></div></div></div></div></div></div></div></div></div><div class=""><div class=""></div></div></div></div></div></pre>

Possible research questions:

* Do positive lyrics stay longer in the charts?
* Are emotionally intense songs more persistent?
* Does lyrical complexity correlate with popularity?

---

### Machine Learning

Another approach is to predict persistence using:

* lyrics
* audio features
* combined feature sets.

Models may include:

* linear regression
* random forests
* gradient boosting.

---

# 10. Expected Contribution

The project illustrates how** ** **text mining methods can enrich traditional music datasets** .

Specifically, it demonstrates:

* how lyrics can be treated as textual data
* how textual features can be constructed
* how lyrical content may influence commercial success.

The project provides a framework for integrating** ** **natural language processing with cultural and entertainment data** .

---

# 11. Pipeline Overview

Complete workflow:

<pre class="overflow-visible! px-0!" data-start="6300" data-end="6592"><div class="relative w-full mt-4 mb-1"><div class=""><div class="relative"><div class="h-full min-h-0 min-w-0"><div class="h-full min-h-0 min-w-0"><div class="border border-token-border-light border-radius-3xl corner-superellipse/1.1 rounded-3xl"><div class="h-full w-full border-radius-3xl bg-token-bg-elevated-secondary corner-superellipse/1.1 overflow-clip rounded-3xl lxnfua_clipPathFallback"><div class="pointer-events-none absolute end-1.5 top-1 z-2 md:end-2 md:top-1"></div><div class="pe-11 pt-3"><div class="relative z-0 flex max-w-full"><div id="code-block-viewer" dir="ltr" class="q9tKkq_viewer cm-editor z-10 light:cm-light dark:cm-light flex h-full w-full flex-col items-stretch ͼk ͼy"><div class="cm-scroller"><div class="cm-content q9tKkq_readonly"><span>Download Spotify chart dataset</span><br/><span>↓</span><br/><span>Filter US songs</span><br/><span>↓</span><br/><span>Compute persistence (days in Top-50)</span><br/><span>↓</span><br/><span>Build song-level table</span><br/><span>↓</span><br/><span>Download lyrics from Genius</span><br/><span>↓</span><br/><span>Remove translation pages</span><br/><span>↓</span><br/><span>Detect English language</span><br/><span>↓</span><br/><span>Clean lyrics</span><br/><span>↓</span><br/><span>Generate text features</span><br/><span>↓</span><br/><span>Merge with Spotify data</span><br/><span>↓</span><br/><span>Statistical analysis</span></div></div></div></div></div></div></div></div></div></div></div></div></pre>
