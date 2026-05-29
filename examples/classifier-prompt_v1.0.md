# contact-role-classifier v1.0

> Worked example for the prompt-iteration-playbook. Synthetic, not tied to any real company.
> Scenario: B2B outbound to small online retailers. Given a contact's job title (and
> optional company note), label how relevant the contact is to a sales conversation.

## Task

You classify a single business contact by their job title into exactly one label:

- `decision_maker`: can approve buying the product (owner, founder, C-level, director or head of the buying function).
- `influencer`: shapes the decision but does not sign off alone (manager, lead, senior individual contributor in the buying function).
- `not_relevant`: unrelated function, too junior to influence, or a role outside the buying center (intern, assistant, unrelated department).

## Rules

1. Judge by the role's authority over the purchase, not by seniority alone.
2. If the title names a function unrelated to the product's buying center, label `not_relevant`.
3. Output only the label, lowercase, no explanation.

## Output

A single token: `decision_maker`, `influencer`, or `not_relevant`.
