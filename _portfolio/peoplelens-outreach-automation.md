---
title: "Automating Personalized Sales Outreach at PeopleLens"
excerpt: "A production pipeline that generates personalized outreach drafts in a specific person's voice, built around a deliberate split between human judgment and machine composition.<br/><img src='/images/500x300.png'>"
collection: portfolio
---

**Tools & Technologies:** Python, Anthropic Claude API, Google Sheets API (gspread, google-auth), HubSpot CRM API (v3 and v4), requests, regex

[View the sanitized source on GitHub](https://github.com/Dqy719/GTM-Outreach-Agent)

## Summary

PeopleLens is an early-stage startup building AI coaching tools for sales organizations. I joined for a 60-day remote internship as a technical GTM intern.

The company goal was straightforward to state and hard to achieve. The sales team was manually writing every piece of outbound LinkedIn and email outreach to sales leaders and executives. This capped output at roughly 35 messages per week, and the team wanted to reach 100. My assignment was to find a way to get there without the messages getting worse.

My personal goal was different. I came into this internship from a computational linguistics background, where the work usually stops at a model that performs well on a held-out set. I wanted to find out what happens after that, when a language system has to live inside somebody else's daily workflow, be maintained by people who do not write code, and survive contact with a stakeholder who judges it by whether it feels right rather than by any metric I could compute. I learned more from the ways that went badly than from the parts that worked.

This writeup covers the outreach draft generator, `generate_drafts.py`, which was the core deliverable of the internship. The version linked in my repository is sanitized: credentials, identifiers, and internal message content have been replaced with placeholders and representative sample data.

## 1. The Problem: Personalization Does Not Scale

The obvious solution was a template with merge fields. I argued against it, and it is worth explaining why, because the argument shaped everything else.

The team's manual drafted messages worked because they were specific and enriched. They referenced a prospect's recent role change, a post they had written, a conference they had attended. A reader could tell the message had been written for them and could not have been sent to anyone else. That property is the entire value of the message, and a merge field template destroys it, because a template is by construction something that could be sent to anyone.

So the design tension was: personalization does not scale, and scale destroys personalization. Rather than accept that tradeoff, I tried to find out whether it was real, by decomposing the task of writing one message into its parts.

Writing an outreach message turns out to be two jobs. The first is **judgment**: deciding what actually matters about this person, out of everything knowable about them. The second is **composition**: turning that judgment into a well-formed message in a consistent voice, with the right level of pressure in the ask, without repeating what earlier messages already said.

These two jobs have very different properties. Judgment is fast for a human who knows the market and slow to fake. Composition is slow for a human, roughly ten minutes per message, and it is exactly the kind of constrained text generation that a large language model is good at. A model handed a LinkedIn profile and asked to pick the hook will confidently pick a mediocre one, because "what makes this person worth talking to" depends on context the model does not have. But a model handed a hook and asked to build a message around it performs well.

This led to the design premise I built the whole system around, which I came to call **augmented authoring**. A human supplies the personalization hook as a short free-text fragment, eight words or fewer, rough and unpolished, possibly with typos. The system supplies everything else: structure, voice, continuity with prior touches, and calibration of the ask. The human remains the bottleneck on the part that requires judgment, but that part now takes fifteen seconds instead of ten minutes.

This was the most consequential decision in the project. It was also the one I had the hardest time communicating, which I will return to in section 6.

## 2. System Architecture

I used the team's existing outreach Google Sheet as the interface rather than building a UI. This was not the technically clean choice. It was the right one, because all the information (conversation histories, contacts, positions, etc.) already lived in that sheet.

The pipeline reads the sheet, classifies each contact into one of three relationship states, builds a prompt specific to that state, calls the Claude API, and writes the draft back to the sheet for human review. Nothing sends automatically. A human always approves before a message reaches a prospect.

The routing logic is the spine of the script:

```python
# Decide message type — three paths
if agreed.lower() in ("y", "yes"):
    # Path 1: warm lead — pull HubSpot history, write warm re-engagement
    msg_type = "warm (agreed to meet)"
    history = get_activity_history(email)
    prompt = build_warm_prompt(contact, voice_dna, history)
elif has_replied.lower() in ("y", "yes"):
    # Path 2: replied but never agreed to meet — LinkedIn DM with CTA tier
    cta_tier = decide_cta_tier(has_replied, comments)
    msg_type = f"linkedin ({cta_tier})"
    prompt = build_linkedin_prompt(contact, voice_dna, cta_tier)
else:
    # Path 3: never replied — whitepaper email
    msg_type = "email (whitepaper)"
    prompt = build_email_prompt(contact, voice_dna)
```

The three paths exist because these are not variations in tone. They are different speech acts.

**Path 1 (warm)** handles a contact who previously agreed to meet and then went quiet. The correct message here acknowledges a shared history and is short, two or three sentences, like texting someone you already know. It needs to know what was actually said before, which is why this path is the only one that hits the CRM.

**Path 2 (replied)** handles a contact who responded but never committed to a meeting. There is a relationship but no momentum. This path writes a LinkedIn DM and has to maintain continuity with the prior touches without recycling the same angle.

**Path 3 (cold)** handles a contact who has never responded to anything. There is no relationship to draw on, so the message leads with content, a whitepaper, a flyer, rather than with an ask.

Collapsing these into one path was the failure mode of every off-the-shelf tool the team evaluated. A warm lead who receives cold outreach framing notices immediately, and the relationship ends up worse than if nothing had been sent.

## 3. Voice DNA

The system writes on behalf of a specific person, so it needed a model of how that person writes.

I collected the team's real past outreach messages, used Claude to extract the recurring patterns across them, and then hand edited the result into a Markdown file that gets loaded into every prompt. I call it the Voice DNA file. It contains tone rules, a four-part message structure, a list of banned constructions (no em dashes, no "I hope this finds you well"), CTA phrasing rules, and a set of real messages grouped by relationship temperature that function as few-shot examples.

Loading it is trivial:

```python
VOICE_DNA_PATH = os.path.join(os.path.dirname(__file__), "voice_dna.md")

def load_voice_dna() -> str:
    with open(VOICE_DNA_PATH, "r", encoding="utf-8") as f:
        return f.read()
```

The design decision worth noting is that the file is deliberately **not** code. Keeping it as a separate Markdown document meant a non-engineer could revise how the system writes without touching the script and without asking me.

## 4. CTA Tiering: Rule-Based Intent Detection

The most linguistically interesting component decides how much pressure to put into the ask. The team had three tiers of call to action, ranging from a soft "open to comparing notes?" to a specific "open to 30 minutes next Wednesday?", and using the wrong one is costly in both directions. Too soft on a hot lead wastes the opening. Too aggressive on a cold one ends the conversation.

I implemented this as a lexicon-based intent classifier over the free-text comments field, where the team records notes from prior interactions:

```python
INTEREST_KEYWORDS = [
    "interested", "interest", "open to", "let's connect", "sounds good",
    "would love", "happy to", "keen", "yes", "sure", "follow up",
    "schedule", "call", "meeting", "chat"
]

def decide_cta_tier(has_replied: str, comments: str) -> str:
    """
    Specific  — replied AND comments suggest prior interest
    Value-led — replied but no clear interest signal
    Soft      — never replied (LinkedIn DM case)
    """
    replied = has_replied.lower() in ("y", "yes")
    if not replied:
        return "soft"
    comments_lower = comments.lower()
    if any(kw in comments_lower for kw in INTEREST_KEYWORDS):
        return "specific"
    return "value"
```

The selected tier is then injected into the prompt as a hard constraint, with the exact CTA sentence the model must end on. The model composes the message but does not get to choose the ask.

I chose rules over asking the LLM to judge intent. The rules are deterministic, auditable, free, and instant. An LLM judgment on the same field would have been more flexible and completely opaque, and it would have added a second API call per contact to a system already bound by write latency.

## 5. Retrieving Conversation History from HubSpot

For warm leads, a generic re-engagement is worse than nothing, so Path 1 pulls the actual prior conversation out of the CRM.

This is harder than it sounds because of how the data is shaped. Emails reach HubSpot as email objects. LinkedIn DMs reach HubSpot through a third-party tool called Birdie, which logs them as communication objects. The two live in different object types, are associated to the contact separately, and have completely different property names. There is no single endpoint that returns "the conversation." You have to fetch both, normalize them, and merge them yourself.

```python
def get_activity_history(email: str, max_items: int = 12) -> str:
    """Pull emails + LinkedIn messages (Birdie) for a contact, newest last."""
    if not email:
        return ""
    contact_id = find_hubspot_contact_id(email)
    if not contact_id:
        return ""

    items = []

    for e in _batch_read("emails", _get_associated_ids(contact_id, "emails"),
                         ["hs_timestamp", "hs_email_subject", "hs_email_text", "hs_email_direction"]):
        p = e.get("properties", {})
        body = _strip_html(p.get("hs_email_text", ""))[:600]
        direction = "sender → contact" if p.get("hs_email_direction") == "EMAIL" else "contact → sender"
        items.append((p.get("hs_timestamp", ""), f"[EMAIL {direction}] {p.get('hs_email_subject','')}: {body}"))

    # ... communications fetched and appended the same way

    items.sort(key=lambda x: x[0])          # oldest → newest
    items = items[-max_items:]              # keep most recent N
    return "\n\n".join(text for _, text in items)
```

Three decisions here are text processing rather than API plumbing. HubSpot returns email bodies as HTML, so `_strip_html` normalizes them to plain text before they enter a prompt, since raw markup wastes context and confuses the model. Ordering is chronological because the model needs to know which turn came last in order to write something that reads as continuation rather than as a fresh start. And truncation happens on two axes, the twelve most recent items and 600 characters each, because prompt context is finite and one long thread will otherwise crowd out the Voice DNA and the task instructions.

Every network call fails soft and returns an empty list rather than raising. A missing history should degrade the message to a hook-based one, not crash a batch run partway through and leave the sheet half written.

## 6. Where the Design Broke: One Flag Carrying Two Facts

The most instructive failure in this project was not a crash. It was a data modeling mistake that I did not recognize as one until it made a feature impossible to build.

The first working version had two paths, driven by a single binary column called `has_replied`. If a contact had replied, they got a LinkedIn DM whose CTA tier was chosen by `decide_cta_tier`. If they had not, they got the whitepaper email. The tier function handled all three levels of ask, including the soft one reserved for people who had never responded.

This was clean, and it was wrong in a way that only surfaced later.

A few weeks in, a requirement appeared that broke it. Some contacts had already agreed to meet and then gone quiet. These are the most valuable contacts in the sheet, and sending them anything cold shaped is worse than sending nothing, because it signals that nobody remembers the conversation they already had.

My first instinct was to repurpose the existing column. Agreeing to meet implies having replied, so `has_replied` could just become `agreed_to_meet` and I would route on that. I made the change and immediately could not distinguish path 2 from path 3. Every contact who had replied but never committed collapsed into the same bucket as contacts who had never responded at all, and started receiving cold whitepaper emails.

The root cause was that I had been treating relationship state as a boolean when it is at minimum ternary. Never replied, replied without commitment, and committed then went quiet are three distinct states. "Did they reply" and "did they agree to meet" are two orthogonal facts about a contact, and one column cannot carry two facts. By overloading it I did not add a state, I traded one away.

The fix was to stop overloading and give each fact its own column, which is what the routing block in section 2 reflects. `agreed_to_meet` and `has_replied` are read independently, and the three paths fall out of the two flags without any single field having to mean two things.

The lesson I took is about the order of reasoning. Before adding a branch, write out the state space that the branch is supposed to partition. If the branches do not partition anything, the schema is wrong and the logic is fine. I had been reasoning about paths, which are code, instead of about states, which are the world. Paths follow from states, and I had it backwards.

There is also a piece of residue I want to be honest about, because it is still visible in the code in section 4. `decide_cta_tier` still accepts `has_replied` as a parameter and still contains a branch returning `"soft"` for non-repliers. After the refactor, that branch is unreachable: `main()` only calls the function when `has_replied` is already Y, so `replied` is always True inside it, and the soft tier survives in `CTA_EXAMPLES` without ever being selected. It is dead code, a fossil of the two-path design, and I did not clean it up before the internship ended. I noticed it while writing this description rather than while writing the code, which is the honest version of the lesson: the refactor was never finished, only made to work.

## 7. Smaller Failures, and the Problem That Was Not Technical
The hardest problem, though, was not technical at all.

The drafts were judged underwhelming, and the initial read from leadership was that the agent's architecture was wrong. This is the moment where it would have been easy to start rebuilding. Instead I went back through the outputs and lined each one up against the input it had been given. The pattern was immediate: the personalization field was almost always filled with the weakest available angle, usually a job title or a career transition, because the teammate filling it in had never been given a priority order for what makes a good hook. The architecture was doing exactly what it was designed to do with the input it was handed.

I documented the diagnosis, proposed a personalization priority hierarchy that ranked personal interests first and job title last, and later built a separate scoring agent to raise input quality at the source.

That episode taught me the thing I most want to keep from this internship. In a system that deliberately splits work between a human and a model, output quality is bounded by whichever side is weaker, and when the output disappoints, everyone's instinct is to blame the model. Being able to show where the loss actually occurred, with evidence rather than assertion, turned out to be more valuable than any code I wrote that month. I also learned that I had not explained the augmented authoring design well enough in the first place. If the stakeholder had understood from the start that personalization was human-selected by design, the input quality problem would have been visible to everyone, not just to me.

## Final Thoughts

At the end of the internship the script was functional end to end. It ran against the live sheet, routed correctly across all three paths, pulled real conversation history out of HubSpot, and produced drafts that the CEO reviewed. It was not adopted into the daily workflow.

I want to be straightforward about that rather than dress it up. The reasons were partly organizational, since a 60-day pilot competing against an existing manual habit is a hard sell in a company with no engineering capacity to maintain what I left behind. And they were partly my own: the input quality problem was diagnosed but not fully solved before I ran out of time. Looking back, my mistake was building the composition layer first because it was the interesting problem, when hook selection was the actual constraint on output quality. If I ran this project again I would build the hook selection tooling first and the drafting second.

The design premise outlived the script. The augmented authoring pattern, human judgment plus machine composition, carried directly into the scoring agent I built afterward, and the Voice DNA extraction approach generalizes to any system that needs to write in a specific person's voice.

What I actually take away is narrower and more useful than "I built an agent." I learned that in applied language work, the model is rarely the hard part. The hard parts are the shape of the data you have to reach through, the humans who have to live with the output, and knowing precisely where your own system stops being trustworthy.

## MSHLT Learning Outcomes

**1. Write, debug, and document readable and efficient code.** The script is a documented single module with a module-level contract describing the expected sheet schema and setup steps, named column constants rather than magic indices, isolated helper functions, and prompt builders separated from I/O and from the API layer. Sections 6 and 7 trace each failure to a root cause rather than to a patch, including a data modeling error that required rethinking the state space rather than the code, and an honest accounting of dead code left behind by an unfinished refactor. Batch reads and soft-failing network calls were efficiency and robustness decisions made against real API constraints.

**2. Select and apply appropriate algorithms and core concepts in HLT.** Lexicon-based intent detection over an unstructured free-text field, chosen over model-based classification for auditability and cost, with an explicit accounting of its failure modes including the negation problem. Text normalization of HTML email bodies before use. Chronological ordering and dual-axis truncation for context window management. Few-shot conditioning and constrained generation through a structured style specification and injected hard constraints.

**3. Apply common tools and libraries by integrating them into real-world workflows.** Four external services integrated into one pipeline running against live business data: the Anthropic API, Google Sheets through service account authentication, the HubSpot CRM across two API versions and multiple object types, and a third-party LinkedIn logging layer whose data model I had to reverse engineer from what it wrote into the CRM.

**4. Demonstrate professional skills.** Scoping and proposing the project independently in a company with no engineering team. Choosing an interface that matched existing team behavior over one that was technically cleaner. Diagnosing a quality complaint to its true cause and presenting the evidence to the CEO. Exercising judgment about what could be published given confidentiality constraints. And communicating an architectural decision to a non-technical stakeholder, which I did not do well enough, and which I would approach differently now.
