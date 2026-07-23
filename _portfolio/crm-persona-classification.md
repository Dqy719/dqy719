---
title: "Classifying CRM Contacts into Sales Personas"
excerpt: "A two-stage classifier that assigns a persona to every contact in a sales CRM: deterministic rule matching first, an LLM for the residual. Built to be cheap, auditable, and honest about what it cannot classify.<br/><img src='/images/500x300.png'>"
collection: portfolio
---

**Tools & Technologies:** Python, pandas, regular expressions, Anthropic Claude API (Haiku), gspread, Google Colab

[View the source on GitHub](https://github.com/Dqy719/REPO-NAME)

## Summary

This project supported a Q2 outreach campaign for the B2B company where I interned. Its CRM contained several thousand contacts whose profile information was inconsistent and whose `Persona` field was mostly blank. To send tailored marketing emails to different ICP groups—such as investors, sales executives, and revenue operations leaders—the company first needed to classify each contact by persona.

I approached this as a text classification problem, using job titles as the primary signal. I designed a two-stage system: a rule-based matcher classified clear cases, while a language model handled the remaining ambiguous titles. I created the persona framework, system architecture, and initial classification rules, then used AI tools to support later improvements and implementation.


## 1. The Problem: Job Titles Are Dirty Text

The CRM’s job-title data was messy and inconsistent. The same role could appear in several formats, while broad terms such as “president,” “data,” or “AI” could mean different things depending on context. Some fields were also blank, corrupted, or filled with placeholder text. As a result, simple keyword matching was not reliable. Stage 1 first normalized the titles and applied context-aware rules before assigning a persona.

## 2. Two-Stage Architecture: Rules First, LLM for the Residual

The design decision that shaped everything was to not reach for the language model first.

Most titles are unambiguous to a human and to a rule. "Chief Revenue Officer" is a CRO. Spending an API call and a model's judgment on it is wasteful, slow, and worst of all opaque: if a rule mislabels a title I can read the rule, but if a model mislabels one I have nothing to point at. So Stage 1 resolves everything it can with deterministic rules, which are free, instant, and auditable. Only the genuine residual, the titles that no rule can confidently place, goes to Stage 2 and the LLM.

This is a cascade. The cheap, transparent classifier runs first and handles the bulk; the expensive, flexible one is reserved for the hard cases the first stage explicitly flags. The handoff is a single boolean column written by Stage 1:

```python
# Flag rows that should go to stage 2 (LLM): blank persona, not junk
df["Needs_Stage2_LLM"] = needs_fill & (df["Persona"] == "") & (df["Persona_Match_Method"] != "junk")
```

Stage 2 reads only the rows where this is true. Junk titles are deliberately excluded from the LLM entirely: there is no point paying a model to classify "Wrong email ID."

## 3. Stage 1: Rule-Based Matching

Every title is first put through normalization, so that the matcher sees one canonical form instead of the many ways the same title can be typed:

```python
def normalize_title(title):
    if pd.isna(title) or str(title).strip() == "":
        return ""
    t = str(title).strip().lower()
    t = t.replace("&", " and ")
    t = re.sub(r"[/,;|]", " ", t)
    t = re.sub(r"\s+", " ", t).strip()
    return t
```

Matching a keyword is then done with word boundaries rather than substring search, because substring matching is a classic source of false positives. Without boundaries, "lead" matches "leadership" and "it" matches "unit." The lookbehind and lookahead here require the keyword to stand as its own token:

```python
def keyword_in_title(keyword, normalized_title):
    kw = keyword.lower().replace("&", " and ")
    kw = re.sub(r"\s+", " ", kw).strip()
    pattern = r"(?<![a-z0-9])" + re.escape(kw) + r"(?![a-z0-9])"
    for m in re.finditer(pattern, normalized_title):
        if kw == "president":
            preceding_words = normalized_title[:m.start()].strip().split(" ")
            if preceding_words and preceding_words[-1] == "vice":
                continue  # "Vice President" is not a CEO/Founder signal
        return True
    return False
```

The `president` branch is the clearest example of why lexical matching alone is not enough. The same token means different things depending on the word before it, so the classifier has to look at local context to disambiguate. This is a small, hard-coded instance of a general principle in text classification: word identity is not word meaning.

The most linguistically interesting rule is combo matching. Rather than enumerate every phrasing of a senior sales title, I match on co-occurrence: a seniority term and the word "sales" appearing within a few tokens of each other, in either order.

```python
def find_term_positions(term, tokens):
    term_tokens = term.split()
    n = len(term_tokens)
    return [i for i in range(len(tokens) - n + 1) if tokens[i:i + n] == term_tokens]


def combo_hit(normalized_title, seniority_terms, role_terms, max_gap):
    tokens = normalized_title.split()
    sen_pos = [p for t in seniority_terms for p in find_term_positions(t, tokens)]
    role_pos = [p for t in role_terms for p in find_term_positions(t, tokens)]
    return any(abs(sp - rp) <= max_gap for sp in sen_pos for rp in role_pos)
```

This single proximity rule covers "Director of Sales," "Senior Vice President of Sales," and "Regional Sales VP" without any of them being written down, because it reasons about the relationship between two words rather than about fixed phrases. It deliberately excludes "manager" and "lead" as seniority markers, holding the persona to a director-and-above bar.

Two more rules handle ambiguity and conflict. Generic words that carry no signal on their own are required to appear alongside a seniority qualifier before they count:

```python
GENERIC_KEYWORDS_NEED_QUALIFIER = {"it", "security", "ai", "data", "analytics", "engineering", "hr", "talent"}
```

And when a title matches more than one persona, a precedence order decides the winner, so that a specific functional signal (Revenue Operations) beats a generic one (the CRO combo) when both fire on the same title.

## 4. Debugging Stage 1

The first version I wrote had a bug that was quietly expensive, and finding it taught me the most in this project.

My original `pick_persona` was supposed to resolve the case where a title matched two or more personas by falling back to the precedence order. It did not. It returned `None` in that situation instead of consulting precedence, which meant every contact with a genuinely senior, multi-signal title, exactly the contacts that mattered most, was silently left unlabeled. Roughly 87 contacts were lost this way, and because the failure was silent, nothing crashed to tell me. I only found it by auditing why so many obviously senior titles had come back blank. The fix was to actually walk the precedence list:

```python
def pick_persona(raw_title):
    matches = match_title(raw_title)
    if not matches:
        return None
    for persona in PERSONA_PRECEDENCE:  # bug fix: precedence is now actually used
        if persona in matches:
            return persona
    return None
```

The second bug came from how pandas handles missing values. My match_method function checked chosen_persona is None, but pandas converted None to NaN, so the condition stopped working. I replaced it with pd.isna(), which correctly detects missing values inside a DataFrame.

```python
if chosen_persona is None or pd.isna(chosen_persona):
    return "none"
```

I also added a junk filter for entries like “Wrong email ID” or “Vendor.” These rows are flagged for cleanup instead of being assigned a persona. A Persona_Match_Method column records how each result was classified, making the output easier to review.

## 5. Stage 2: LLM Classification for the Residual

Stage 2 takes only the rows Stage 1 flagged and asks Claude Haiku to place them, using persona definitions I wrote to constrain the model to the same ten categories. I chose Haiku because this is a short, high-volume classification task where a small fast model is the right tool, and batched 25 titles per call to keep the number of requests reasonable across roughly 1,900 candidate rows.

Before spending any API calls, Stage 2 filters out inputs an LLM cannot reliably handle. Blank titles are skipped. So are garbled ones, and the garbage detector is where the linguistic care shows:

```python
def is_garbled(title):
    if is_blank(title):
        return False
    s = str(title)
    if "\ufffd" in s:  # unicode replacement character
        return True
    if re.search(r"[\x00-\x08\x0b\x0c\x0e-\x1f]", s):  # stray control chars
        return True
    if re.search(r"â€|[ÃÂ][€™¢œ¦°...]", s):  # classic utf-8/windows-1252 mojibake
        return True
    non_ascii = sum(1 for c in s if ord(c) > 127)
    has_cjk_or_hangul = bool(re.search(r"[\u4e00-\u9fff\u3040-\u30ff\uac00-\ud7af]", s))
    if len(s) > 0 and non_ascii / len(s) > 0.5 and not has_cjk_or_hangul:
        return True
    return False
```

The subtle requirement here is that a high proportion of non-ASCII characters must not be treated as corruption on its own, because a legitimate Chinese, Japanese, Korean, or accented-Latin title would trip that wire. The detector explicitly exempts CJK and Hangul ranges, so it flags true mojibake while leaving real non-English titles alone. Getting this wrong would mean silently discarding every international contact.

The model is instructed to return strict JSON and to answer "None" rather than force a fit when a title matches no persona. The call layer is defensive in the ways a batch job against an external API has to be: exponential-backoff retries, tolerance for the model wrapping its JSON in code fences, and a permanent-failure path that leaves rows blank rather than crashing the whole run.

```python
for attempt in range(4):
    try:
        resp = client.messages.create(...)
        text = resp.content[0].text.strip()
        text = re.sub(r"^```(json)?|```$", "", text.strip(), flags=re.MULTILINE).strip()
        parsed = json.loads(text)
        return {int(item["id"]): item["persona"] for item in parsed}
    except Exception as e:
        wait = 2 ** attempt
        time.sleep(wait)
```

## 6. Honest Limitations

I want to be explicit about what this system does not do, because the gaps were deliberate decisions rather than oversights, and documenting them is part of the work.

Two large groups of contacts are intentionally left unclassified. COO, CFO, CMO, and CPO titles, around 75 rows, do not fit any of the ten personas as scoped, so they are left blank rather than forced somewhere wrong. Individual-contributor sales roles, Account Executive, SDR, BDR, and the like, around 177 rows, likewise have no persona, because the scheme targets director-level-and-up sales titles by design. Mislabeling these would be worse than leaving them blank, since a wrong persona sends a contact down the wrong downstream path.

The other honest caveat is that the Stage 2 persona definitions are inferred from my Stage 1 keyword lists, not copied from an authoritative persona document, and the quality of every LLM classification depends entirely on those definitions being right. I flagged this directly in the code so that whoever runs it next knows to review the definitions before trusting the output.

## 7. Collaboration and Tooling

The design of this system is mine: the ten-persona scheme, the decision to lead with rules and reserve the LLM for the residual, and the first working version of the Stage 1 matcher, which I wrote and then found the precedence bug in. From there I used AI tooling to help implement the later improvements, the combo proximity matching and the Stage 2 scaffolding, working from my own diagnoses and design intent. I am describing this plainly because it is how the work actually happened, and because knowing how to direct these tools, decide what they should build, and audit what they produce is itself part of working in this field now. The judgment about what to build and why was the part I owned, and it is the part that matters.

## Final Thoughts

At the end of the internship, Stage 1 was complete and had been run against the live contact data. Stage 2 was fully implemented and validated on a small sample of test titles, confirming that it produced correct classifications end to end, but I ran out of time before executing it on the full set of roughly 1,900 flagged rows. The honest status is a finished, tested two-stage system whose second stage had not yet had its full production run.

If I were to continue, the immediate next step is exactly that full run, followed by a manual audit of the `Persona_Match_Method` column to measure how often each stage was right. The natural extension is to feed the persona labels into the outreach segmentation downstream, closing the loop between who a contact is and what they should be sent.

What I take from this project is that most of the difficulty in classifying real text is not the classifier. It is the normalization, the disambiguation, the ambiguity control, and the discipline to leave a thing unlabeled rather than label it wrong. The model in Stage 2 was the smallest part of the work.

## MSHLT Learning Outcomes

**1. Write, debug, and document readable and efficient code.** A two-stage pipeline with normalization, isolated matching functions, and an auditable output column, plus the two debugging narratives in section 4: a silent precedence bug that cost 87 contacts, and a pandas type-coercion trap where `is None` stopped working inside a DataFrame.

**2. Select and apply appropriate algorithms and core concepts in HLT.** Text normalization, word-boundary matching over naive substring search, context-sensitive disambiguation of ambiguous tokens, proximity-based co-occurrence matching as a generalization over fixed phrases, precedence-based conflict resolution, and a rule-plus-LLM cascade that reserves the expensive classifier for the residual.

**3. Apply common tools and libraries in real-world workflows.** pandas for the data pipeline, regular expressions for the matching engine, the Anthropic API with batching and backoff for the second stage, and gspread to read live CRM data out of Google Sheets.

**4. Demonstrate professional skills.** Designing a system around cost and auditability rather than reaching for the most powerful tool, documenting deliberate limitations instead of hiding them, giving an honest account of AI-assisted development and my own role in it, and building for a successor by recording where each label came from and what still needs review.
