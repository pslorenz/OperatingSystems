# Chapter 9: The AI-Assisted Practitioner

**You come in with:** a working knowledge of networking from Ch01-Ch08. You've been practicing the verification habit throughout the unit. You've seen AI explain things correctly and seen it confidently invent things.
**You leave with:** durable working patterns for using AI as a copilot in IT and security work. The discipline of guarded believability internalized. The ability to recognize when AI is helping and when it's misleading. A short list of what to never share with an LLM.

**Time:** 60 to 75 minutes. Less command-line work than other chapters; more thinking.

**Security+ alignment:** Domain 5.2 (risk assessment, including emerging technology risks). Indirectly: AI tools are used in modern SOC operations, vulnerability management, and threat intelligence (Domains 4.4, 4.5). The exam doesn't test AI usage practices directly; this chapter is for working practice, not the test.

---

## Why this chapter exists

Every other chapter in this unit had a moment where AI showed up: explaining a Wireshark filter, decoding a DMARC record, walking through a firewall rule. That was deliberate. AI is part of how working IT and security practitioners now operate. Skipping AI in a 2026 networking course would produce graduates whose first job already includes tools the curriculum ignored.

But AI in IT work is genuinely different from AI as a chat toy or a writing assistant. The consequences of AI mistakes in IT range from "you wasted ten minutes" to "you applied a destructive change to production." That cost gradient is what makes the working patterns matter.

This chapter is the audience-defining chapter for this unit. M1's audience-defining chapter was Linux Artifacts (the chapter that made the security focus explicit). M2's was Windows Artifacts. M3's is this one: the chapter that establishes how a modern practitioner uses AI without being captured by it.

The framing is copilot, not oracle. Junior colleague whose work you check, not authority whose answers you trust. The discipline is verification.

---

## What AI is good at, in this domain

Honest accounting. AI is genuinely useful for several IT and security tasks:

### Explaining unfamiliar output

You hit `tcpdump` output that looks weird. You read a firewall log entry you don't recognize. You see an event ID in Windows Security log you've never investigated.

LLMs are good at explaining well-documented things. They've been trained on RFCs, vendor docs, blog posts, Stack Overflow. For "what does this output mean," they're often a faster path than searching docs yourself.

The verification: cross-check the explanation against an authoritative source. If the LLM says "this TCP flag indicates X," verify against RFC 9293. If it says "this Windows Event ID means Y," verify against Microsoft Learn. Most of the time the explanation is correct; sometimes it's confidently wrong in ways that matter.

### Drafting first-pass scripts

You need a one-off PowerShell script to find files modified in the last week. You need a bash one-liner to extract IPs from a log. You need a Python script to parse a CSV and reformat it.

LLMs are good at producing first-pass code for well-defined tasks. The output is usually close to working, sometimes works first try, sometimes needs adjustment.

The verification: read the code line by line before running it. Understand what it does. Test on a sample, not on production data. If you can't read and understand the code, you can't take responsibility for what it does.

### Translating concepts between platforms

You know how to do something on Linux and need the Windows equivalent. You know iptables and need to express the same rule in Windows Defender Firewall. You know Bash and need the PowerShell equivalent.

LLMs are good at translation. Linux-to-Windows, Bash-to-PowerShell, vendor-A-syntax-to-vendor-B-syntax. They've been trained on enough cross-platform content to handle these well.

The verification: test the translated command on a non-production system. The translation is usually correct in spirit; sometimes specific flags or option names differ.

### Generating documentation drafts

You need to write up what your firewall rules do. You need to document the network architecture you just designed. You need to write a runbook for an operational task.

LLMs are good at producing documentation drafts when you give them the technical details. They're better at the prose than they are at the technical accuracy, so the workflow is: provide the facts, ask for prose, edit for accuracy.

The verification: never ship LLM-generated documentation without reading and editing. Read every claim against your own knowledge of what's actually deployed.

### Brainstorming and rubber-duck debugging

You're stuck on a problem. You can't figure out why something isn't working. You want a list of possibilities to investigate.

LLMs are good at generating broad lists. Ask "what are the common causes of X" and you get a useful enumeration. Some items will be relevant to your situation; some won't.

The verification: treat the list as candidates, not answers. Each item gets evaluated against your specific circumstances.

---

## What AI is bad at

Equally honest. AI fails predictably in several IT and security contexts:

### Recently changed details

LLM training data has a cutoff. Vendor product changes, CVE details from after the cutoff, recently published RFCs, current pricing, current version-specific behavior all get reported confidently with information that's stale or wrong.

This bites particularly in:

- **CVE details:** "What was CVE-2024-XXXXX?" gets either a confident wrong answer or "I don't know."
- **Vendor product features:** "What does the latest version of pfSense support?" misses recent additions.
- **Cloud platform specifics:** AWS, Azure, GCP change constantly. AI knowledge lags.

The verification: for anything time-sensitive, go to the authoritative source. NIST NVD for CVEs, vendor docs for product features, official cloud platform docs for cloud specifics.

### Specific commands with subtle wrongness

"Run this command to do X" sometimes produces commands that are almost right but subtly wrong. Common patterns:

- Flag names that sound right but don't exist for that command (`-Recurse` on a Linux command, `--all` on a PowerShell cmdlet).
- Syntax that worked in a previous version but changed.
- Plausible-sounding but nonexistent subcommands.

The verification: read the man page or `Get-Help` output. Don't trust LLM-generated commands without checking.

### Specific log patterns

When you ask "what does this specific log message mean," the LLM sometimes invents an explanation that sounds reasonable. The danger is if the actual cause is different from the invented explanation, you go down the wrong investigation path.

The verification: search the actual log message text against the vendor's documentation or community resources. If you can't find a reference, the LLM's confidence about the meaning should be discounted.

### Anything involving secrets

LLMs don't know your environment. When asked about your specific systems, they either decline (if they recognize the question is environment-specific) or invent (if they don't).

The verification: never ask an LLM about specifics of your environment as if the LLM knows them. Provide the specifics yourself; ask the LLM to reason about what you've provided.

### Critical thinking

LLMs don't reliably push back when an idea is bad. Asked "should I do X," they often validate X even when X is risky.

The verification: when you ask for validation, you don't get it. You get acknowledgment phrased as validation. Real critical review comes from someone (or something) that has both context and the will to disagree.

### Subtle security implications

LLMs can describe security concepts but often miss the second-order implications. "Should I disable this protection" gets answered narrowly without considering downstream effects.

The verification: for security-relevant changes, check against vendor recommendations and security baselines (CIS Benchmarks, NIST SP 800-series). The LLM is one input; the authoritative sources are the decision-makers.

---

## The verification habit

Throughout the unit you've seen the verification habit applied. This section is the explicit pattern.

### Why verification matters specifically in IT

Mistakes in IT have specific shapes:

- A command that "works on the lab" might fail or do something different in production.
- A configuration suggestion might be sound for the version you don't have.
- A security recommendation might miss the threat model your environment actually faces.

The cost of getting it wrong ranges from minor (you waste time) to major (you cause an outage or security incident). The cost asymmetry is what makes verification worth the time.

### What verification looks like

For different kinds of LLM output, verification has different shapes:

**For explanations:** check against the authoritative source for that domain. RFCs for protocols. Vendor docs for product behavior. NIST or vendor advisories for vulnerabilities.

**For commands:** read the man page or `Get-Help`. Confirm the flags exist and do what's claimed. Test on a non-production system before applying anywhere important.

**For code:** read line by line. Trace the logic. Consider the edge cases (empty input, unusual characters, race conditions). Test on sample data.

**For recommendations:** cross-check against established frameworks (CIS, NIST, vendor security guides). The LLM is one input among several.

**For factual claims:** if the claim is time-sensitive (current product features, current threat landscape), go to a current source. If the claim is older and stable (how a protocol works), the LLM is probably right but verify if the consequences of being wrong are large.

### Practical verification for an LLM-suggested PowerShell

LLM gives you:

```powershell
Get-WinEvent -FilterHashtable @{LogName='Security'; Id=4625; StartTime=(Get-Date).AddDays(-7)} |
    Group-Object -Property @{Expression={$_.Properties[5].Value}} -NoElement |
    Sort-Object Count -Descending
```

It claims this counts failed logon attempts by source user, last 7 days.

Verification:

1. **Read the cmdlet syntax.** `Get-WinEvent -FilterHashtable` is real. `Group-Object -Property` accepts script blocks. `Sort-Object` is fine.

2. **Read the script logic.** Filtering Event 4625 (failed logon) is correct. The filter for last 7 days is correct.

3. **Check the property index.** `$_.Properties[5]` is the username for Event 4625? Verify by looking at one event:

```powershell
Get-WinEvent -FilterHashtable @{LogName='Security'; Id=4625} -MaxEvents 1 |
    Select-Object -ExpandProperty Properties
```

Read the output. If `[5]` is actually the username, the LLM was right. If it's something else, the script will group by the wrong property.

4. **Test on real data.** Run the original script. Verify the output makes sense (usernames in the result column, counts that match what you'd expect).

This took 60 seconds. It catches the case where the LLM hallucinated property index 5 when the actual index is 1.

---

## What never to share with an LLM

Brief and direct. These have privacy or security implications that go beyond "I should be careful."

### Secrets

Passwords. API keys. SSH private keys. TLS private keys. Connection strings with embedded credentials. Personal access tokens. Anything in `~/.aws/credentials` or similar.

The reasons:

- LLMs may train on inputs (depends on the service and your settings).
- Logs and analytics may capture inputs.
- The LLM service is a third party; data flows there leave your control.

If you have to discuss code with secrets, redact first. Replace with placeholders. Confirm the redacted version doesn't leak the original.

### Customer or user data

PII (personally identifiable information). Health records. Financial details. Anything covered by privacy regulations (GDPR, HIPAA, etc.).

Same reasons as secrets, plus the additional issue that you may be violating contractual or regulatory obligations by sharing the data outside your environment.

If you have to discuss data, use synthetic examples. The LLM doesn't need real data to help you with the structure or processing logic.

### Internal architectural details

Things that would be useful to an attacker. Internal IP ranges plus specific server roles. Credentials systems and how they're configured. Specific firewall rules with destination IPs. The structure of your authentication flow.

You don't have to be paranoid about this; talking generally about your environment is fine. The line is "would this specific information help an attacker if it leaked." If yes, generalize before asking.

### Active investigation details

If you're investigating a security incident, the details are sensitive. The attacker's IPs, the systems they touched, the indicators you've found, the response you're planning. These shouldn't go to a third-party LLM service.

Use a local LLM (one running on infrastructure you control) for incident-specific work, or stick to general questions ("how do I investigate persistence on Linux") that don't include specifics.

---

## A working pattern: AI as junior colleague

The mental model that works best is: AI is a junior colleague who knows a lot, has confident opinions, and doesn't always realize when they're wrong. You wouldn't deploy production changes based on a junior's first suggestion without review. Treat AI the same way.

Concretely:

**For work where you're learning:** AI is the explanation companion. You read something, ask the LLM to explain, then verify against the source. The LLM accelerates your learning by pointing you at concepts; the source confirms.

**For work where you have expertise:** AI is the first-draft generator. You know what you want; the LLM gets you 80% of the way; you finish the last 20% with your domain knowledge.

**For work where stakes are high:** AI suggestions are inputs, not answers. You evaluate them with the same scrutiny you'd apply to any other input from someone less experienced than you.

**For work where you're deeply uncertain:** AI is least reliable here. If you can't tell whether a suggestion is good, you can't tell whether the LLM's suggestion is good either. This is when you reach for human expertise (a more senior colleague, vendor support, a community of practice).

---

## A practical exercise

Pick three real tasks you might do as a junior network or security practitioner:

1. **Investigation task.** "I see a suspicious DNS query in our logs to a domain I don't recognize. How do I investigate?"

2. **Configuration task.** "I need to write a pfSense rule that blocks outbound connections to a specific IP from a specific subnet but allows everything else."

3. **Documentation task.** "Write a runbook for the procedure of replacing a failed switch in our office."

For each:

1. Ask an LLM the question. Capture the response.

2. Apply the verification habit:
   - For task 1: which steps does it suggest are universally correct? Which depend on your environment? Which references it cites are real (verify by checking)?
   - For task 2: does the rule it suggests actually do what's described? What edge cases would break it? What would you change before applying?
   - For task 3: is the runbook correct and complete? What's missing? What would you remove because it doesn't fit your environment?

3. Write up your verification. Where was the LLM helpful? Where was it wrong or misleading? What's the workflow you'd use if you had to do this for real?

The exercise is not about catching the LLM. It's about practicing the discipline of using AI without trusting it.

---

## Common stumbling blocks

> **The LLM keeps giving me different answers to the same question.**
> Yes. LLMs aren't deterministic; the same prompt can produce different outputs. This is informative: if the answer changes when you ask twice, the model isn't confident. Probe for the variation deliberately when you need to know whether an answer is stable.

> **The LLM is more confident than I am, so I tend to defer.**
> Confidence is not accuracy. The LLM's tone is calibrated for plausibility, not truth. Your domain expertise plus skepticism is more reliable than the LLM's confidence.

> **I don't know the authoritative source to verify against.**
> Then you're not yet in a position to evaluate the LLM's answer. This is a learning opportunity: find the authoritative source for your domain. RFCs for protocols, vendor docs for products, NIST for security frameworks. Bookmarking these is the start of being a working practitioner in the area.

> **My environment doesn't have a local LLM option for sensitive work.**
> Then you don't use LLMs for sensitive work. The constraint is real. Most organizations are working out their policy on this; some allow specific approved providers, some require local models. Find your organization's policy before sharing anything sensitive.

> **The LLM's first answer is wrong but if I push back it gives me the right answer.**
> Sometimes. Sometimes if you push back, it gives you a different wrong answer that agrees with your pushback. The model wants to satisfy you; "I think you're wrong about X" can produce "you're right, the answer is actually Y" without Y being correct either. Don't take agreement as confirmation.

> **I asked the LLM for sources and it gave me URLs that don't exist.**
> Common failure mode. LLMs hallucinate citations confidently. Always click through and verify URLs before treating them as sources. If you can't find the source, the claim is suspect.

---

## What this gets you

After this chapter:

- You have working patterns for using AI as a copilot in IT and security work.
- You can recognize the kinds of tasks where AI is useful and the kinds where it misleads.
- You have a verification habit that catches the common failure modes.
- You know what never to share with a third-party LLM.
- You can work productively with AI tools without being captured by their confidence.

The verification habit is the most durable piece. It transfers to all kinds of authority, not just AI. Vendor sales pitches, blog post recommendations, vendor docs that turn out to be outdated, advice from senior colleagues who haven't kept current. The discipline of "I check this before I act on it" is what makes a senior practitioner senior.

---

## What's next

Chapter 10 is the network hardening capstone. CIS-aligned host firewall hardening across Linux and Windows, with a verification script. The portfolio diff that demonstrates "I took a default install and made it materially more secure, with documented evidence." Same shape as M1 Ch12 and M2 Ch12. The unit closes here.
