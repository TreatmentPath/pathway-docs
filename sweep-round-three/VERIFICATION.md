# Round-three sweep — verification pass

Same discipline applied to the sweep that the sweep applied to the code: re-derive every
number, audit every label, and look hardest at my own claims.

---

## 1. Every measured claim REPRODUCES — 12 of 12

Re-run against the database and source, compared to what was recorded:

| claim | recorded | re-measured |
|---|---|---|
| #91 phones shared by >1 recall patient | 3,138 | **3,138** |
| #91 emails shared by >1 recall patient | 2,884 | **2,884** |
| #92 live Persons with >1 nurture | 44 | **44** |
| #92 live Persons with >1 intake | 497 | **497** |
| #93 live Persons with >1 email channel | 3,008 | **3,008** |
| #93 `ContactChannel._meta.ordering` | `[]` | **`[]`** |
| #94 `duplicate_contact` clusters | 252 | **252** |
| #94 clusters holding 2+ distinct humans | 9 | **9** |
| #95 activity logs person-vs-patient | 685 / 9 mismatched | **685 / 9** |
| G1 practice gate | 101 calls / 65 gates | **101 / 65** |
| G7 dentally `except→Response` | 90 (83/3/2) | **90 (83/3/2)** |
| G13 helper uses / hand-built joins | 63 / 82 | **63 / 82** |

## 2. Every `✅ VERIFIED` label carries evidence

All 20 items marked VERIFIED/REAL contain either an executed-output block or a measured
count. No bare assertions. `#93` is deliberately labelled "VERIFIED MECHANISM" rather than
VERIFIED because the two functions agree today; that hedge is still correct.

---

## 3. ⚠️ TWO OF MY OWN CLAIMS OVERSTATED THEIR IMPACT

The same error I refuted eight times in the agents' output. Both mechanisms are real; both
impact statements were wrong.

### G12 — "renders 'None Smith' on the public booking page" — NOT REACHABLE

I proved `User.first_name` is nullable and that 2 users have NULL, then described patients
reading "None Smith" as their practitioner's name. Checked properly:

```
users with NULL first_name: [(48,'jb@gmail.com'), (338,'e2e-forms@example.com')]
…of those, practitioner on an OnlineBookingHold: 0
total OnlineBookingHold rows: 0
```

**The holds table is EMPTY**, and both NULL users are test accounts — one explicitly named
`e2e-forms@example.com`. `views.py:684` never executes today and would not hit a NULL if it
did. The unguarded f-string is still worth fixing (it is one `or ''` away from correct, and
`full_name()` exists) but no patient has seen anything.

### C2 / C7 — "`Patient.__str__` renders 'John None'" — TRUE, but the rows are DEMO DATA

I reported "1 Patient of 60,536; 1 Intake of 3,966" as live exposure. The actual rows:

```
Patient 139394 'Ethan', practice 9 'practicedemo'   ->  str() = 'Ethan None'
Intake  3888   'Test',  practice 13                  ->  practice 13 is the TEST practice
```

Both are in a demo practice and the test practice. The `__str__` defect is real and
reproduces exactly as recorded — but framing it as production exposure was wrong.

**Corrected severity for both:** latent code defects worth fixing cheaply as part of the
`full_name()` consolidation, NOT live incidents. This is the "0 rows today" category the
main audit already uses for several findings.

---

## 4. ⚠️ COVERAGE — the sweep is INCOMPLETE

| track | done | still owed |
|---|---|---|
| identity (haiku) | 11 dirs | `Tasks/`, `medicalHistory/`, EmailServiceGo packages, 2 frontend dirs |
| general (haiku) | 7 dirs | everything below |
| codex | 6 dirs | `marketingBroadcast/`, `Notes/`, `Documents/`, `automations/`, `dataQuality/` |

**Patient-relevant apps NEVER scanned by any track:**

`Tasks` · `medicalHistory` · `patientDocuments` · `patient_accounts` · `teamChat` ·
`UserAuthentication` · `Invoices` · `Labs` · `dedupe_audit` · `practiceConnection`

Three of those matter more than the rest:

- **`patient_accounts`** and **`patientDocuments`** — both hold records per patient and have
  never been examined for the CARRY-vs-DISCOVER pattern.
- **`teamChat`** — already implicated in **G13** as the source of the second, privacy-rule-
  free `get_user_display_name`.
- **`UserAuthentication`** — holds 2 of the 11 client-IP copies (C12).

**So the correct statement of status is: the findings that exist are sound and reproduce;
the sweep itself has covered roughly two-thirds of the relevant ground.**

---

## 5. What this pass did NOT check

- Whether the ~26 unscanned apps contain findings (that is the remaining sweep, not a
  verification gap).
- Whether the 18 URL builders (G11) have drifted — flagged as unmeasured when recorded, and
  still unmeasured.
- Which of G13's 83 sites reach a guest-visible surface — flagged when recorded, still open.
- The 4 codex findings listed as "pending verification" in NEW-FINDINGS.md.
