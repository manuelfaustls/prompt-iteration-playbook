# contact-role-classifier v1.1

> Change vs v1.0: ONE change. Added an example-anchor pair to rule 1 to fix a subtle
> miss (senior-sounding titles that are actually individual contributors with no authority).
> The bare rule wording alone did not land the distinction, so a positive case plus a
> contrast negative case was added. See the run-log for the measured lift.

## Task

You classify a single business contact by their job title into exactly one label:

- `decision_maker`: can approve buying the product (owner, founder, C-level, director or head of the buying function).
- `influencer`: shapes the decision but does not sign off alone (manager, lead, senior individual contributor in the buying function).
- `not_relevant`: unrelated function, too junior to influence, or a role outside the buying center (intern, assistant, unrelated department).

## Rules

1. Judge by the role's authority over the purchase, not by seniority alone.
   - Example: "Head of Growth" at a small retailer usually owns the buying decision for growth tooling, so `decision_maker`.
   - Contrast: "Senior Growth Analyst" sounds senior but is an individual contributor who reports up, so `influencer`, not `decision_maker`.
2. If the title names a function unrelated to the product's buying center, label `not_relevant`.
3. Output only the label, lowercase, no explanation.

## Output

A single token: `decision_maker`, `influencer`, or `not_relevant`.
