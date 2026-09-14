# Patient-identity audit — the 90 findings ALREADY FOUND AND FIXED

Do NOT report any of these again. A new finding must be a DISTINCT mechanism,
not another instance of one of these unless the instance is in code none of
these touched — and if so, say which number it resembles and why it is separate.

#1: the name-split mismatch has already made duplicate Persons
#2: the recall list opens the wrong person's record
#3: "Add family member" deliberately merges two humans
#4: a treatment plan can be reassigned to another practice's patient
#5: clinical AI summaries are shared between same-named patients
#6: family lookup leaks patients across practices
#7: a lookup computes a phone key the writer never writes
#8: the Dormant recall tab loses the contact for multi-word names
#9: `Person.resolve` called without a date of birth
#10: `+GB…` phone numbers, so the SMS silently never sends
#11: the merge suggester compares phones as raw strings
#12: a lead can be stripped of all contact details on update
#13: three more places still prefer Dentally's broken phone field
#14: a workflow patient lookup is case-sensitive on email
#15: assigning a bad or foreign practitioner returns success
#16: the Dentally bridge scores matches across practices
#17: Activity History shows another patient's clinical notes  [CONFIRMED]
#18: online booking files the appointment under a family member  [CONFIRMED]
#19: the booking phone fallback can never match  [CONFIRMED by execution]
#20: booking payment creates a Patient by splitting at the first space  [CONFIRMED]
#21: two more first-space splitters feeding `Person.resolve`  [CONFIRMED]
#22: clinical notes and letters accept any practice's patient  [CONFIRMED]
#23: Tasks scopes the user FKs, not the patient FKs  [CONFIRMED]
#24: Appointments accepts any patient and any clinician  [CONFIRMED]
#25: `by_contact` returns the oldest family member's log  [CONFIRMED]
#26: medical-history portal downgrades DOB verification to name-matching  [CONFIRMED, one agent claim WRONG]
#27: Pattern E: `patient_name` on a Task is read-only, PATCH silently no-ops  [CONFIRMED]
#28: Pattern E: booking asks "new or existing patient?" and never reads it  [CONFIRMED]
#29: hand-rolled phone normalisation in the consent SMS path  [CONFIRMED]
#30: medical-history submission never checks that verification happened  [MINE, found while checking #26]
#31: the verification code is not bound to the plan it unlocks  [CONFIRMED]
#32: the practice check fails open  [CONFIRMED]
#33: online booking creates a Patient with no Person at all  [CONFIRMED]
#34: `_match_patient` can match an archived patient  [CONFIRMED]
#35: `patient_activities` never validates that `patient_id` is in the caller's practice  [CONFIRMED]
#36: day-list history counts cross practice boundaries  [CONFIRMED]
#37: one marketing profile represents an arbitrary member of a fused Person  [CONFIRMED]
#38: the call agent chooses and persists the first human on a shared phone  [CONFIRMED]
#39: the call-agent DOB field is accepted and then ignored  [CONFIRMED]
#40: ActivityLog create/update accepts foreign Persons and objects  [CONFIRMED by execution]
#41: Go call-agent Intakes are inserted outside the identity graph  [CONFIRMED]
#42: recall sync re-splits Dentally's correct first and last names  [CONFIRMED]
#43: consent management crosses practices on read and write  [CONFIRMED by execution]
#44: active Telnyx phone lookup computes a different key from the identity writer  [CONFIRMED]
#45: one patient can hold several AI summaries under name variants
#46: an outbound SMS on a shared family line was recorded against nobody
#47: a CARRY implemented as a DISCOVER mints duplicate Persons
#48: custom letter routes disclosed another practice's clinical letter
#49: custom note-by-patient routes disclosed a former practice's clinical note
#50: Intake/Nurture API fields selected arbitrary siblings from a fused Person
#51: live edit surfaced re-split stored names at the first space
#52: inbound email trusts an unauthenticated practice ID
#53: email-template preview is scoped; send is not
#54: patient CSV import treated a shared family contact as proof of one arbitrary patient
#55: phone-only plan association ORed away its practice/match predicate
#56: patient-account charges accepted a practitioner from another practice
#57: consent signing accepts another patient's plan/appointment as context
#58: legacy conversation reader lowercases the key, then compares case-sensitively
#59: invoice update exposed a global practitioner FK
#60: Patient's preferred clinician FKs were globally writable
#61: marketing webhook ignores its exact recipient and picks the first Person on the email
#62: consent-to-Dentally sync chose one arbitrary Patient on a fused Person
#63: booking OTP prefill reveals whichever family member is first
#64: Inbox sends carried no recipient id, so attribution could never fire
#65: "Add to Nurture" discarded the patient and forced a re-DISCOVER
#66: Inbox tasks were filed under a name string, not a patient
#67: practice switch left the previous practice's WhatsApp config live
#68: the frontend fabricated "Unknown" as a first name
#69: Day List and Recall leaked patient PII across a practice switch
#70: clinical notes were read AND written by patient NAME
#71: universal search opened a different human
#72: Add Open/Active Plan demanded a patient id, then discarded it
#73: a task created from a Nurture row attached to an arbitrary sibling
#74: call logging dropped every identifier it was holding
#75: call-log patient attribution picked an arbitrary sibling
#76: the shared-line guard did not cover its own fallthrough
#77: a third producer of placeholder names, this one backend
#78: websocket consumers authenticated the user but never authorised the practice
#79: a conversation titled with the WRONG family member  [was #48]
#80: outbound email bypassed the sandbox, and was unattributed  [was #49]
#81: automated sends were filed against nobody  [was #50]
#82: unauthenticated Twilio media stream could create identity in ANY practice
#83: a workflow node's config overrode the execution's authenticated practice
#84: Dentally import overwrote an arbitrary same-name relative
#85: call-agent PersonChannel insert always fails
#86: call agent collapsed a fused Person to its first Patient
#87: Telnyx assistant read every family member's appointments off a shared phone
#88: every three-digit international dialling code was corrupted
#89: Go wrote the literal name "Unknown"
#90: the AI email-intake parser invented names that became identities
