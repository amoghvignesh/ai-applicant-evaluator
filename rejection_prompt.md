# Rejection Feedback Prompt

Used by the `Prepare Rejection Email Request` → `Call Claude API (Rejection Body)` nodes.

- **Model:** `claude-sonnet-4-5`
- **Max tokens:** 500
- **Endpoint:** `POST https://api.anthropic.com/v1/messages`
- **Output:** plain text (one paragraph). Inserted into the rejection email template.

This prompt only runs when the evaluation step returns `decision: "FAIL"`. Its
job is narrow: produce the specific feedback paragraph that goes into the
candidate's rejection email. Greeting, signature, and overall structure are
templated outside this call.

---

## Prompt (single user message; no system block)

Variables interpolated at call time are shown as `${...}`.

```text
You are writing the feedback paragraph for a rejection email to a job applicant on behalf of the hiring team. The email wrapper, greeting, and closing are already templated; your job is to write only the specific feedback content.

Constraints:
- 2-3 sentences total
- Reference one specific concrete thing from their materials (a project, a claim, a repo) — not generic
- Use "we" throughout — never "I" or "the AI" or any first-person singular
- Respectful and direct, not corporate-fake-warm
- Frame as a fit gap, not a deficiency in the candidate
- Plain prose, no bullet points, no greeting, no signature, no signoff

The paragraph should:
- State the specific gap or signal that drove the decision
- Ground it in evidence from their actual application
- Avoid generic recruiter language

What to avoid:
- "We've decided to move forward with other candidates"
- "While your background is impressive..."
- "Best of luck in your future endeavors"
- First-person singular ("I noticed", "I reviewed")
- Mentioning AI, automated systems, or screening tools
- Apologizing or hedging excessively

Input data:

Candidate name: ${applicantName}
Specific rejection reason: ${evaluation.specific_rejection_reason}
Concerns: ${JSON.stringify(evaluation.concerns)}
Inconsistencies: ${JSON.stringify(evaluation.inconsistencies)}

Return ONLY the feedback paragraph text. No greeting, no closing, no signature.
```

---