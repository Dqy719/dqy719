---
title: "Automating Personalized Sales Outreach at PeopleLens"
excerpt: "A production pipeline that generates personalized outreach drafts by splitting the work between human judgment and machine composition. Python, Claude API, HubSpot CRM, Google Sheets."
collection: portfolio
permalink: /portfolio/peoplelens-outreach-automation/
date: 2026-07-01
---


Tools & Technologies: Python, Anthropic Claude API, Google Sheets API (gspread, google-auth), HubSpot CRM API (v3 and v4), requests, regex
Summary
PeopleLens is an early-stage startup building AI coaching tools for sales organizations. I joined for a 60-day remote internship as a technical GTM intern, reporting directly to the CEO. The company was small enough that there was no engineering team behind me and no existing automation to inherit, which meant that whatever I proposed, I also had to build.
The company goal was straightforward to state and hard to achieve. The CEO was personally writing every piece of outbound LinkedIn and email outreach to sales leaders and executives. This capped output at roughly 35 messages per week, and the team wanted to reach 100. My assignment was to find a way to get there without the messages getting worse.
My personal goal was different. I came into this internship from a computational linguistics background, where the work usually stops at a model that performs well on a held-out set. I wanted to find out what happens after that, when a language system has to live inside somebody else's daily workflow, be maintained by people who do not write code, and survive contact with a stakeholder who judges it by whether it feels right rather than by any metric I could compute. I learned more from the ways that went badly than from the parts that worked.
This writeup covers the outreach draft generator, generate_drafts.py, which was the core deliverable of the internship. The version linked in my repository is sanitized: credentials, identifiers, and internal message content have been replaced with placeholders and representative sample data.
1. The Problem: Personalization Does Not Scale
The obvious solution was a template with merge fields. I argued against it, and it is worth explaining why, because the argument shaped everything else.
The CEO's messages worked because they were specific. They referenced a prospect's recent role change, a post they had written, a conference they had attended. A reader could tell the message had been written for them and could not have been sent to anyone else. That property is the entire value of the message, and a merge field template destroys it, because a template is by construction something that could be sent to anyone.
So the design tension was: personalization does not scale, and scale destroys personalization. Rather than accept that tradeoff, I tried to find out whether it was real, by decomposing the task of writing one message into its parts.
Writing an outreach message turns out to be two jobs. The first is judgment: deciding what actually matters about this person, out of everything knowable about them. The second is composition: turning that judgment into a well-formed message in a consistent voice, with the right level of pressure in the ask, without repeating what earlier messages already said.
These two jobs have very different properties. Judgment is fast for a human who knows the market and slow to fake. Composition is slow for a human, roughly ten minutes per message, and it is exactly the kind of constrained text generation that a large language model is good at. A model handed a LinkedIn profile and asked to pick the hook will confidently pick a mediocre one, because "what makes this person worth talking to" depends on context the model does not have. But a model handed a hook and asked to build a message around it performs well.
This led to the design premise I built the whole system around, which I came to call augmented authoring. A human supplies the personalization hook as a short free-text fragment, eight words or fewer, rough and unpolished, possibly with typos. The system supplies everything else: structure, voice, continuity with prior touches, and calibration of the ask. The human remains the bottleneck on the part that requires judgment, but that part now takes fifteen seconds instead of ten minutes.
This was the most consequential decision in the project. It was also the one I had the hardest time communicating, which I will return to in section 6.
2. System Architecture
I used the team's existing outreach Google Sheet as the interface rather than building a UI. This was not the technically clean choice. It was the right one, because the CEO already lived in that sheet, and asking a small team to adopt a new tool during a 60-day pilot is a reliable way to make sure the pilot fails.
The pipeline reads the sheet, classifies each contact into one of three relationship states, builds a prompt specific to that state, calls the Claude API, and writes the draft back to the sheet for human review. Nothing sends automatically. A human always approves before a message reaches a prospect.
The routing logic is the spine of the script:
python# Decide message type — three paths
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
The three paths exist because these are not variations in tone. They are different speech acts.
Path 1 (warm) handles a contact who previously agreed to meet and then went quiet. The correct message here acknowledges a shared history and is short, two or three sentences, like texting someone you already know. It needs to know what was actually said before, which is why this path is the only one that hits the CRM.
Path 2 (replied) handles a contact who responded but never committed to a meeting. There is a relationship but no momentum. This path writes a LinkedIn DM and has to maintain continuity with the three prior touches without recycling the same angle.
Path 3 (cold) handles a contact who has never responded to anything. There is no relationship to draw on, so the message leads with content, a whitepaper, rather than with an ask.
Collapsing these into one path was the failure mode of every off-the-shelf tool the team evaluated. A warm lead who receives cold outreach framing notices immediately, and the relationship ends up worse than if nothing had been sent.
3. Voice DNA
The system writes on behalf of a specific person, so it needed a model of how that person writes.
I collected the CEO's real past outreach messages, used Claude to extract the recurring patterns across them, and then hand edited the result into a Markdown file that gets loaded into every prompt. I call it the Voice DNA file. It contains tone rules, a four-part message structure, a list of banned constructions (no em dashes, no "I hope this finds you well"), CTA phrasing rules, and a set of real messages grouped by relationship temperature that function as few-shot examples.
Loading it is trivial:
pythonVOICE_DNA_PATH = os.path.join(os.path.dirname(__file__), "voice_dna.md")

def load_voice_dna() -> str:
    with open(VOICE_DNA_PATH, "r", encoding="utf-8") as f:
        return f.read()
The design decision worth noting is that the file is deliberately not code. I could have put all of it in Python string literals. Keeping it as a separate Markdown document meant a non-engineer could revise how the system writes without touching the script and without asking me. That mattered more than it sounds. Voice was the part of the system most likely to need ongoing tuning, and I was the person most likely to leave.
4. CTA Tiering: Rule-Based Intent Detection
The most linguistically interesting component decides how much pressure to put into the ask. The team had three tiers of call to action, ranging from a soft "open to comparing notes?" to a specific "open to 30 minutes next Wednesday?", and using the wrong one is costly in both directions. Too soft on a hot lead wastes the opening. Too aggressive on a cold one ends the conversation.
I implemented this as a lexicon-based intent classifier over the free-text comments field, where the team records notes from prior interactions:
pythonINTEREST_KEYWORDS = [
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
The selected tier is then injected into the prompt as a hard constraint, with the exact CTA sentence the model must end on. The model composes the message but does not get to choose the ask.
I chose rules over asking the LLM to judge intent, and this was a deliberate decision rather than a shortcut. The rules are deterministic, auditable, free, and instant. When the CEO asked why a particular contact received an aggressive ask, I could point at a line of code. An LLM judgment on the same field would have been more flexible and completely opaque, and it would have added a second API call per contact to a system already bound by write latency.
The tradeoff is real, and I did not solve it. The lexicon has no negation scoping, so "not interested" contains the substring "interest" and misroutes to the most aggressive tier, which is the worst possible failure. Matching is purely lexical, so any paraphrase outside the list is invisible to it. I mitigated this by making human review mandatory before send, which caught misroutes before they reached a prospect, but mitigation is not a fix. The right solution is negation handling plus a labeled set of real comment strings to evaluate against, and I did not have enough labeled data inside the internship window to justify building it.
I am documenting this failure mode rather than hiding it because knowing where a classifier breaks is the difference between a system you can trust and one you merely hope about.
5. Retrieving Conversation History from HubSpot
For warm leads, a generic re-engagement is worse than nothing, so Path 1 pulls the actual prior conversation out of the CRM.
This is harder than it sounds because of how the data is shaped. Emails reach HubSpot as email objects. LinkedIn DMs reach HubSpot through a third-party tool called Birdie, which logs them as communication objects. The two live in different object types, are associated to the contact separately, and have completely different property names. There is no single endpoint that returns "the conversation." You have to fetch both, normalize them, and merge them yourself.
pythondef get_activity_history(email: str, max_items: int = 12) -> str:
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
Three decisions here are text processing rather than API plumbing. HubSpot returns email bodies as HTML, so _strip_html normalizes them to plain text before they enter a prompt, since raw markup wastes context and confuses the model. Ordering is chronological because the model needs to know which turn came last in order to write something that reads as continuation rather than as a fresh start. And truncation happens on two axes, the twelve most recent items and 600 characters each, because prompt context is finite and one long thread will otherwise crowd out the Voice DNA and the task instructions.
Every network call fails soft and returns an empty list rather than raising. A missing history should degrade the message to a hook-based one, not crash a batch run partway through and leave the sheet half written.
6. Debugging, and the Problem That Was Not Technical
The first failures were unglamorous. The script died with a UnicodeDecodeError while reading the Voice DNA file, because Python on Windows defaults to cp1252 and the file contained characters outside that encoding. Adding encoding="utf-8" to the open call fixed it, and it taught me to be explicit about encoding at every file boundary rather than trusting platform defaults.
Then GOOGLE_SA_JSON resolved to None and raised a TypeError deep inside the credentials loader, because I was reading it from an environment variable that was not set in the context I was actually running from. Then WorksheetNotFound, because the tab name in my config did not match the tab name in the sheet. Both errors surfaced far from their causes. The general lesson is that configuration errors fail late and misleadingly, so configuration should be validated at startup rather than at first use.
I also made a security mistake worth documenting. I exposed an Anthropic API key while sharing code to get help debugging. I revoked it immediately, but the correct habit is to never have live credentials sitting in a file you might share. This is also why the public version of this script has every credential, identifier, and piece of internal message content replaced with placeholders.
The hardest problem, though, was not technical at all.
The drafts were judged underwhelming, and the initial read from leadership was that the agent's architecture was wrong. This is the moment where it would have been easy to start rebuilding. Instead I went back through the outputs and lined each one up against the input it had been given. The pattern was immediate: the personalization field was almost always filled with the weakest available angle, usually a job title or a career transition, because the teammate filling it in had never been given a priority order for what makes a good hook. The architecture was doing exactly what it was designed to do with the input it was handed.
I documented the diagnosis, proposed a personalization priority hierarchy that ranked personal interests first and job title last, and later built a separate scoring agent to raise input quality at the source.
That episode taught me the thing I most want to keep from this internship. In a system that deliberately splits work between a human and a model, output quality is bounded by whichever side is weaker, and when the output disappoints, everyone's instinct is to blame the model. Being able to show where the loss actually occurred, with evidence rather than assertion, turned out to be more valuable than any code I wrote that month. I also learned that I had not explained the augmented authoring design well enough in the first place. If the stakeholder had understood from the start that personalization was human-selected by design, the input quality problem would have been visible to everyone, not just to me.
Final Thoughts
At the end of the internship the script was functional end to end. It ran against the live sheet, routed correctly across all three paths, pulled real conversation history out of HubSpot, and produced drafts that the CEO reviewed. It was not adopted into the daily workflow.
I want to be straightforward about that rather than dress it up. The reasons were partly organizational, since a 60-day pilot competing against an existing manual habit is a hard sell in a company with no engineering capacity to maintain what I left behind. And they were partly my own: the input quality problem was diagnosed but not fully solved before I ran out of time. Looking back, my mistake was building the composition layer first because it was the interesting problem, when hook selection was the actual constraint on output quality. If I ran this project again I would build the hook selection tooling first and the drafting second.
The design premise outlived the script. The augmented authoring pattern, human judgment plus machine composition, carried directly into the scoring agent I built afterward, and the Voice DNA extraction approach generalizes to any system that needs to write in a specific person's voice.
What I actually take away is narrower and more useful than "I built an agent." I learned that in applied language work, the model is rarely the hard part. The hard parts are the shape of the data you have to reach through, the humans who have to live with the output, and knowing precisely where your own system stops being trustworthy.
MSHLT Learning Outcomes
1. Write, debug, and document readable and efficient code. The script is a documented single module with a module-level contract describing the expected sheet schema and setup steps, named column constants rather than magic indices, isolated helper functions, and prompt builders separated from I/O and from the API layer. Section 6 traces each failure to a root cause rather than to a patch. Batch reads and soft-failing network calls were efficiency and robustness decisions made against real API constraints.
2. Select and apply appropriate algorithms and core concepts in HLT. Lexicon-based intent detection over an unstructured free-text field, chosen over model-based classification for auditability and cost, with an explicit accounting of its failure modes including the negation problem. Text normalization of HTML email bodies before use. Chronological ordering and dual-axis truncation for context window management. Few-shot conditioning and constrained generation through a structured style specification and injected hard constraints.
3. Apply common tools and libraries by integrating them into real-world workflows. Four external services integrated into one pipeline running against live business data: the Anthropic API, Google Sheets through service account authentication, the HubSpot CRM across two API versions and multiple object types, and a third-party LinkedIn logging layer whose data model I had to reverse engineer from what it wrote into the CRM.
4. Demonstrate professional skills. Scoping and proposing the project independently in a company with no engineering team. Choosing an interface that matched existing team behavior over one that was technically cleaner. Diagnosing a quality complaint to its true cause and presenting the evidence to the CEO. Exercising judgment about what could be published given confidentiality constraints. And communicating an architectural decision to a non-technical stakeholder, which I did not do well enough, and which I would approach differently now.
