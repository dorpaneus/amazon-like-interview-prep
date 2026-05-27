# Day 13 — Behavioral Intensive (Sun May 10)

The technical scaffolding is built. Today is **rehearsal**. Four remaining LP stories, then the meta-skill: how to actually run a 5-hour Amazon loop without losing energy, judgment, or your story bank.

Tomorrow (Day 14) is light review and sleep. **Today is the last day to do real work** before the loop. Don't peak too early — finish the stories, run a mock loop, then stop.

---

## Morning Block (3h) — The remaining four LPs

Story-writing format: prompt, what a strong version looks like, common failure modes, then you write your STAR. Target: complete first drafts of all four by lunch.

### 13A. Frugality (45 min) — Story #13

> *"Accomplish more with less. Constraints breed resourcefulness, self-sufficiency, and invention. There are no extra points for growing headcount, budget size, or fixed expense."*

**Prompt**:

> "Tell me about a time you accomplished a goal with significantly fewer resources than were originally proposed."

**What a strong version looks like for SRE/DevOps**:

- Cost optimization with a number: cut S3 storage by X%, reduced CloudWatch log volume by N TB/month saving $K, downsized a fleet by Y instances after profiling.
- Built an internal alternative that replaced a vendor: in-house monitoring that retired a $200K/year contract, a small Python tool that replaced a paid SaaS.
- Did the same on-call coverage with a smaller rotation by automating away the toil.
- Avoided a buildout: argued against new infrastructure that wasn't needed, used spare capacity instead.

**Failure modes**:

- "I worked hard / I stayed late" → that's effort, not Frugality. The LP is about *resource efficiency*, not heroics.
- "We had a tight deadline so we cut scope" → not Frugality either; that's prioritization.
- No number → Frugality stories live and die on the magnitude. "Saved money" doesn't land. "Reduced monthly spend from $120K to $85K — 29% — by switching the log aggregation pipeline" lands.

**The bonus structure** that lands well: *resources you chose not to consume*. "The proposed solution required two more engineers and a $300K/year service. I shipped the same outcome alone in six weeks using [open-source tool / existing internal capability]."

Write your STAR now. Time it at 2-3 min out loud.

### 13B. Hire and Develop the Best (45 min) — Story #14

> *"Raise the performance bar with every hire and promotion. Recognize exceptional talent and willingly move them throughout the organization. Develop leaders and take seriously our role in coaching others."*

**Prompt** (interviewers usually pick one):

> "Tell me about a time you helped develop someone — coaching, mentoring, or a tough piece of feedback."
>
> *or*
>
> "Tell me about a hire you made that significantly raised the team's bar — and how you knew."

**What a strong version looks like**:

For **develop**:

- Specific person, specific gap, specific intervention, specific outcome
- Coaching that led to a promotion, role change, or capability they took with them
- A piece of hard feedback that landed and changed behavior — bonus if they later thanked you for it
- Took someone from junior to mid, or mid to senior, with concrete moves you made

For **hire**:

- You drove the decision (or strongly influenced it)
- A specific bar-raising trait you identified — not just "they were great"
- How you tested for it (the bar-raiser mindset)
- Their impact post-hire, measurable if possible

**Failure modes**:

- "I'm a mentor" → personality statement, not a story.
- "I told them to be better" → not coaching; tell what you actually *did*.
- Hire story without a measurable bar raise → just "we hired someone good." Not the LP.
- Story where you "developed" someone by giving them more work → not development; that's delegation.

**Senior-flavored angle**: develop *yourself* by raising the bar in adjacent dimensions. Hiring someone stronger than you in their domain (without ego), then learning from them.

Write your STAR. 2-3 min.

### 13C. Strive to be Earth's Best Employer (45 min) — Story #15

> *"Work every day to create a safer, more productive, higher performing, more diverse, and more just work environment. Lead with empathy, have fun at work, and make it easy for others to have fun. Ask themselves: are my fellow employees growing? Are they empowered? Are they ready for what's next?"*

**Prompt**:

> "Tell me about a time you made your team or workplace better — for someone other than yourself."

**This is one of the two newest LPs (added 2021).** Many candidates fumble it because they read "best employer" and think it requires HR-shaped stories. It doesn't. For an SRE/DevOps role, the highest-landing version is **operational humanity**:

**What a strong version looks like**:

- Reduced on-call burden meaningfully — eliminated false-positive pages, fixed the root cause of a recurring incident class, redistributed coverage fairly.
- Built or fought for psychological safety — created a blameless postmortem culture, defended a junior who was being blamed, made it safe to ask "stupid" questions.
- Made the team more inclusive in concrete ways — changed an interview process that was excluding good candidates, sponsored someone whose work wasn't being noticed, fixed a meeting structure that was silencing certain voices.
- Improved a process that was causing burnout — a deployment ritual, a fire-drill culture, a "hero work" reward structure.

**Failure modes**:

- "I believe in psychological safety" → values statement, not a story.
- "I care about diversity" → same. The LP demands action.
- "I organized team lunches" → too small; doesn't move the bar.
- "I implemented a wellness program" → only if you led it and there's a real outcome.

**The single highest-landing pattern**: a story about **changing on-call** — the most operationally human thing in SRE. "Our on-call rotation was burning people out — three engineers had quit citing it. I [did specific things], which dropped page volume by N% and after-hours pages by M%. Retention on the team improved, and we made it a model that two other teams adopted."

Write your STAR. 2-3 min.

### 13D. Success and Scale Bring Broad Responsibility (45 min) — Story #16

> *"Started in a garage, but we're not there anymore. We are big, we impact the world. Be humble and thoughtful about even the secondary effects of our actions. We must begin each day with a determination to make better, do better, and be better."*

**Prompt**:

> "Tell me about a time you considered the second-order or downstream impacts of a decision — and what you did differently because of them."

**The other newest LP (2021).** Hardest to write a great story for. The shape interviewers are listening for: *you saw beyond the immediate scope of your work*.

**What a strong version looks like**:

- Pushed back on a launch because of a security/privacy/abuse risk that the launching team hadn't fully considered. Bonus if you stayed and helped solve it instead of just blocking.
- Designed something with a deliberate "leave it better than you found it" choice — refactored adjacent code, fixed a documentation gap, raised an issue that wasn't yours.
- Caught an abuse or fairness concern: a metric being gamed, a feature with disparate impact, a default that hurt new users while helping power users.
- Considered the on-call cost to *other* teams: built something that would have created paging for a different team, redesigned it to avoid that.
- Thought about scale: this works for our 100 customers but will create [specific bad thing] at 10,000.

**Failure modes**:

- "I care about doing the right thing" → values statement.
- "We avoided technical debt" → too generic and doesn't capture the "broad responsibility" angle.
- A story where the broad impact is hypothetical / never tested → weakens the punch.

**The "humble and thoughtful" angle**: a strong moment is when you *noticed* a secondary effect that wasn't your job to consider. Show that judgment, then the action you took because of it.

Write your STAR. 2-3 min.

---

## Midday Block (2.5h) — Story bank, mock loop, mechanics

### Story-bank matrix (30 min)

You should now have **16 stories** (one per LP). The senior move is to recognize that each strong story tags to **2–3 LPs**. Build a matrix so during the interview you can quickly pick the best story for the LP being probed — including whichever you haven't told yet.

Make a sheet like this:

| # | Story (1-line summary) | Primary LP | Secondary LPs |
|---|---|---|---|
| 1 | The on-call ownership story | Ownership | Earn Trust, Dive Deep |
| 2 | The slow-query investigation | Dive Deep | Insist on Highest Standards, Bias for Action |
| 3 | The post-mortem follow-through | Insist on Highest Standards | Ownership, Earn Trust |
| ... | ... | ... | ... |

**Why this matters**: a Bar Raiser will probe with a follow-up like "tell me about *another* time…" If you only have one story per LP, you're stuck. With the matrix, you know which other stories tagged to that LP you can pull.

**Also map "I was wrong" stories.** Interviewers love a story where you made a mistake and recovered. Tag those across Earn Trust, Are Right A Lot (the "I sought disconfirming evidence too late" variant), and Dive Deep.

### Interview mechanics (45 min)

**The opening line** for every behavioral. Have a default:

> "Let me tell you about the time when..."

or

> "This reminds me of a project at \[company\] where..."

Concrete > abstract. Don't start with "I generally" or "I believe" — that's a values statement, not a story.

**STAR pacing**:

| Beat | Time | What you say |
|---|---|---|
| Situation | ~20s | "I was at \[company\], we had \[concrete situation\]." |
| Task | ~20s | "My role was \[your specific scope\]. The challenge was \[specific\]." |
| Action | ~90s | "I did A. Then B. Then C. The hard part was \[X\] and I handled it by \[Y\]." |
| Result | ~30s | "We \[outcome with number\]. Looking back, I'd \[reflection — one sentence\]." |

Total: ~2.5 min. The Action block is the bulk. **Most candidates rush Action and over-describe Situation** — invert that ratio.

**Use "I" not "we"**. Interviewers are evaluating *you*. Saying "we" repeatedly forces them to ask "what did *you* do specifically?" — wasted clock. Acknowledge the team briefly, then talk about your own actions.

**Handling interruptions and follow-ups**:

The interviewer cuts in with "wait — tell me more about how you decided X." That's a *signal*: they want depth on that part. Don't re-narrate the whole story.

- Answer their specific question fully.
- Then a brief bridge: "and that led me to \[next beat\]" — steers back to the arc.
- If they don't want the rest of the story, they'll cut you off again. Let them drive.

**The "what would you do differently" question** is coming. Every behavioral has a high chance of it. Have a *real* answer:

- Not "nothing, it was great." That signals lack of self-reflection.
- Not "I'd do everything differently." That undermines the story.
- The sweet spot: one specific thing, with a one-sentence reason. "I'd have brought in \[stakeholder\] two weeks earlier — when I finally did, it unstuck the whole project, and I should have seen that sooner."

**Questions YOU ask at the end** — this matters more than candidates realize. **Different question for each interviewer.** Rotate from a list. Have ~6 prepared. Examples:

- For an engineer: "What does a typical week look like for someone in this role on your team? What surprised you about it when you joined?"
- For the hiring manager: "What does success in the first 6 months look like? What would I be doing that would make you say 'great hire'?"
- For the bar raiser (if you can spot them): "How does the team think about technical disagreements — how do you decide when to push and when to commit?"
- For anyone: "What's the most challenging thing about Managed Operations specifically — what's harder than people expect?"
- For anyone: "How does the European Sovereign Cloud team interact with the broader AWS service teams? Where does most of the cross-team coordination happen?"
- For anyone: "What's something you wish someone had told you before you joined?"

**Don't ask**:

- "What's the culture like?" — generic, signals you don't know what to ask.
- Anything answered on the careers page.
- "Do you offer remote work / what's the comp?" — save for the recruiter.

**Energy management across a 4-6 interview loop**:

- **Water before each interview.** Even if you're not thirsty.
- **Bathroom between every interview** if you can. Reset your body.
- **Don't replay the previous interview** during the break. It's over. Reset.
- **Eat something at lunch.** Loop fatigue is real and food helps. Don't go big — sandwich-sized.
- **Take a beat before each one.** 60 seconds of slow breathing before they walk in.
- The loop ends and you'll feel like you bombed. **Everyone feels that.** It's not predictive.

### Mock loop run (60 min)

Set a timer. Play this as if it's the loop.

**Round 1 — Dive Deep (5 min)**:

> "Tell me about a time you investigated a problem that others had given up on, or where the obvious answer turned out to be wrong."

Speak out loud. Time the response. Don't read your notes.

**Round 2 — Ownership (5 min)**:

> "Tell me about a time you took on something outside your job description because it was the right thing for the team or the company."

**Round 3 — Customer Obsession (5 min)**:

> "Tell me about a time you went beyond what was asked to address a customer's underlying need."

**Round 4 — Bias for Action (5 min)**:

> "Tell me about a time you had to make a decision quickly without having all the information you wanted."

**Round 5 — Have Backbone (5 min)**:

> "Tell me about a time you disagreed with a decision your team or leader made."

**Round 6 — Frugality (5 min)**:

> "Tell me about a time you accomplished a goal with significantly fewer resources than were originally proposed."

After each one, give yourself 30 seconds to score:

- Did I open strong? (No "uh, I guess one time…")
- Did I use "I" not "we"?
- Did I land a number in the Result?
- Did I stay under 3 minutes?
- If interrupted with "tell me more about X," do I have depth?

**Then a technical scenario** (5–10 min): pick one of these and walk it out loud as if to an interviewer.

- "Walk me through what happens when I load a website in a browser."
- "A process is hung at 0% CPU. Walk me through your investigation."
- "We're getting OOM kills on a fleet host. What do you do?"
- "Calculate the TCP buffer size you'd set for a 10 Gbps × 100 ms RTT path, and tell me why."

**Total mock loop time**: ~50 min. Stop. Don't run a second one today.

### Cross-LP weak spots (15 min)

After the mock, identify your two weakest stories. The ones where you stumbled, ran long, or felt fuzzy. **Tomorrow you rehearse only those two** plus the headline technical walks. Don't touch the strong ones tomorrow — they're done.

---

## Afternoon Block (1.5h) — Pre-interview ritual + final framing

### The 36-hour countdown

**Today (Sunday) evening**:

- Stop technical study by dinner.
- Light review of your story matrix only.
- Lay out interview-day clothes / setup if it's remote.
- **Sleep early.** Yes — Sunday night, not Monday night. Two good nights matter more than one.

**Tomorrow (Monday — Day 14)**:

- Light day. No new material. Re-read your reference docs at low intensity.
- Re-read the JD once. Re-read the LPs once.
- One quiet rehearsal of your two weakest stories.
- One walkthrough of the "what happens when I load a website" recital.
- One walkthrough of the "I'd start with the USE method..." performance opener.
- **Stop by mid-afternoon.** Take a walk. Eat dinner early. Sleep at your usual time or slightly earlier — *not* dramatically earlier, which can backfire.

**Tuesday (Day 15 — interview day)**:

- Eat breakfast you've eaten before. Not the morning to experiment.
- Coffee at your normal time. Not extra.
- Re-read the JD one last time, ~20 min before. Then close it.
- 5-10 minutes before: stop reviewing. Sit quietly. Breathe slowly.
- If remote: test your audio/video 30 min before — not 5 min.

### Mental framing for the loop

A few things to internalize:

1. **The interviewers want you to succeed.** They're not adversarial. They want to hire someone. Help them say yes.
2. **You will feel like you bombed at least one of the interviews.** Almost every successful candidate reports this. It's not predictive.
3. **The Bar Raiser is one of them.** Don't try to identify them; treat everyone like the Bar Raiser. Same energy, same depth, same rigor.
4. **"I don't know" is a complete sentence — when followed by a plan.** "I don't know that specific tool — here's how I'd find out: I'd start with X, then check Y." Better than guessing.
5. **The first 30 seconds of each interview set the tone.** Smile, eye contact (or camera contact), warm intro, brief. Then let them drive.
6. **You're allowed to take a beat.** "Let me think for a moment" — say it explicitly, then think. 3-5 seconds of quiet thinking is much better than rambling while you find the answer.
7. **Don't apologize.** "I'm sorry, that's a long answer" / "Sorry, I'm rambling." Just be concise next time. Apologies eat time and signal nervousness.

### Common late-stage mistakes

- **Over-rehearsing on Monday and arriving stale on Tuesday.** Trust your prep. You've done two solid weeks.
- **New caffeine or new food on Tuesday.** Stick to known patterns.
- **Trying to memorize stories verbatim.** Memorize the *beats*, speak naturally.
- **Treating the loop like a test rather than a conversation.** It is fundamentally a conversation. Be a person they'd want to work with.
- **Spending the lunch break refining stories.** That's anxiety, not preparation. Eat. Stretch. Reset.
- **Asking the recruiter for feedback immediately.** They won't give it. Wait for the official decision.

### A note on outcomes

You won't know until after the loop how it went. Reading the room isn't reliable in this format. **Your job during the interview is to do your best work, not to interpret the interviewers' reactions.**

If you don't get this one, you'll have:

- A complete, rigorous SRE/DevOps prep package
- 16 polished STAR stories
- A clear sense of where your gaps are
- The vocabulary and reasoning patterns that work in senior loops

That's not a consolation prize — it's a base that compounds for every interview you ever do after this.

### Optional final exercise (30 min)

If you have energy left and want to use it well: pick **one** of your strongest stories. Record yourself telling it on your phone. Listen back. You'll notice things you wouldn't notice live. Don't re-record more than once.

Then stop. Close the laptop. Go for a walk.

---

## Tomorrow (Day 14, Mon May 11) — light review only

Your only assignments tomorrow:

1. Re-read the JD once.
2. Re-read the LP list once.
3. Re-read your two weakest stories. Speak each one out loud once. Don't perfect them.
4. One slow recital of "what happens when I load a website."
5. One slow recital of "the USE method opening."
6. Glance at the connectivity-debugging tree and the OOM kill workflow.
7. Confirm your interview logistics — time, link or address, who you're meeting.
8. **Sleep at your usual time. Not early. Not late.**

That's it. Day 14 isn't a study day. It's a *consolidation and rest* day.
