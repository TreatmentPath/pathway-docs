# Online booking identity sweep

Scope: `TreatmentPathBackend/TreatmentPath/onlineBooking/` only. I read source
code and the identity helpers it directly invokes; I did **not** execute the
application, tests, migrations, database queries, or HTTP requests.

## Findings

None.

I excluded the mechanisms already listed in `ALREADY-FIXED-INDEX.md`, including
the booking family-member match (**#18**), phone-key mismatch (**#19**),
first-space payment split (**#20**), bare-Patient creation (**#33**), and
shared-email OTP prefill (**#63**). The remaining active matching code scopes
patients to `hold.practice`, canonicalizes phone lookup through
`ContactChannel.lookup_key`, and returns no patient when the name-filtered
candidate set is ambiguous. The abandoned-session writer delegates Person
assignment to the existing Intake signal; its no-DOB resolver path resembles
already-fixed **#9**, so it is not reported again.
