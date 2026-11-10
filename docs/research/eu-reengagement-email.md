# EU rules for re-engagement email (Reminders)

Research for issue #8 (map: issue #1, "Notifications and Reminders"). Researched 2026-11-10.

**Not legal advice.** This summarises primary sources so the spec can make a defensible default. Before launch, a German lawyer (IT/competition law) should sign off on the Reminder copy and the checkout notice, especially the open points marked **Unsettled**.

Vocabulary follows `GLOSSARY.md`. A **Reminder** is an email-only nudge to an inactive Student to come back. A **Notification** is an in-app inbox item that may also be emailed.

## Answer in brief

1. **Treat Reminders as advertising (Werbung / direct marketing).** German courts read "Werbung" broadly. A satisfaction survey attached to an invoice email for an item already bought counted as advertising (BGH VI ZR 225/17). A "come back to your course" email aims at retention and further sales, so it will almost certainly be treated the same way. The safe default is to give Reminders the full set of marketing-email rules.
2. **The default rule is prior express consent.** UWG §7(2) Nr. 2 says email advertising without the addressee's prior express consent is always an unreasonable nuisance. ePrivacy Art. 13(1) says the same for the EU.
3. **The existing-customer exemption (UWG §7(3), ePrivacy Art. 13(2)) can cover Reminders to Students who bought a course.** It only applies if all four conditions hold, and the fourth is the one products usually miss. When the address is collected (at checkout or sign-up), the customer must be told clearly that they can object at any time at no extra cost. Every Reminder must repeat that notice and include a working opt-out. Without the notice at collection, the exemption arguably fails for that customer.
4. **Double opt-in is not required by statute when the exemption applies.** For consent-based sending (anyone outside the exemption), German practice is to use double opt-in. This is not a statutory rule, but the advertiser has to prove consent (GDPR Art. 7(1)), and the BGH treats double opt-in as the accepted way to do that (I ZR 164/09).
5. **Unsubscribe.** The law requires an easy, free objection route in every Reminder (UWG §7(2) Nr. 3c, §7(3) Nr. 4; GDPR Art. 21(2)–(4)). Gmail and Yahoo require bulk senders to support RFC 8058 one-click unsubscribe on marketing mail (`List-Unsubscribe` with an HTTPS URL plus `List-Unsubscribe-Post: List-Unsubscribe=One-Click`, both covered by DKIM), show a visible unsubscribe link in the body, and honour unsubscribes within 2 days (48 hours).
6. **Transactional Notification emails are a different category.** A reply-to-your-Question email is a service message triggered by the Student's own activity, so it is not advertising and needs neither consent nor a one-click header. That only holds while it stays free of promotion: adding promotional content to such a mail turns it into advertising (BGH VI ZR 134/15). Notification emails should still offer a preferences link, because of map scope and user expectations rather than the law.

## Sources and findings

### 1. ePrivacy Directive 2002/58/EC, Art. 13 (as amended by 2009/136/EC)

Source: [EUR-Lex consolidated text](https://eur-lex.europa.eu/legal-content/EN/TXT/HTML/?uri=CELEX:02002L0058-20091219)

- Art. 13(1): email "for the purposes of direct marketing may be allowed only in respect of subscribers or users who have given their prior consent."
- Art. 13(2), the existing-customer exception: where a person "obtains from its customers their electronic contact details for electronic mail, in the context of the sale of a product or a service … the same natural or legal person may use these electronic contact details for direct marketing of its own similar products or services provided that customers clearly and distinctly are given the opportunity to object, free of charge and in an easy manner, … at the time of their collection and on the occasion of each message".
- Art. 13(4) prohibits marketing email that disguises the sender or lacks "a valid address to which the recipient may send a request that such communications cease".

This is a directive. In Germany it applies through UWG §7 (below).

### 2. German UWG §7 (the binding national rule for a German operator)

Source: [gesetze-im-internet.de, §7 UWG](https://www.gesetze-im-internet.de/uwg_2004/__7.html)

- §7(2) Nr. 2: an unreasonable nuisance is always assumed "bei Werbung unter Verwendung … elektronischer Post, ohne dass eine vorherige ausdrückliche Einwilligung des Adressaten vorliegt".
- §7(2) Nr. 3: separately prohibited are emails that hide the sender's identity, breach §6(1) DDG (commercial communication must be identifiable as such), or have "keine gültige Adresse …, an die der Empfänger eine Aufforderung zur Einstellung solcher Nachrichten richten kann".
- §7(3) is the existing-customer exemption. It applies only if **all four** conditions hold:
  1. The business obtained the address from the customer "im Zusammenhang mit dem Verkauf einer Ware oder Dienstleistung".
  2. The address is used for direct advertising of "eigene ähnliche Waren oder Dienstleistungen".
  3. The customer has not objected.
  4. The customer "bei Erhebung der Adresse und bei jeder Verwendung klar und deutlich darauf hingewiesen wird, dass er der Verwendung jederzeit widersprechen kann", at no cost beyond basic transmission costs.

What this means for Cadence:
- **Condition 1.** A Student who bought a course at checkout clearly qualifies. A Student who got their access some other way, for example a **Seat assigned by a Team Admin**, did not give their address "in connection with a sale" to them. **Unsettled**, but probably not covered, so these Students need consent.
- **Condition 2.** A nudge to continue the course they already bought, or to look at closely related courses, fits "own similar products" well. Cross-selling unrelated products would stretch it. German courts have traditionally read "ähnlich" narrowly (roughly: the same typical use or need as the purchased item).
- **Condition 4.** This needs a **product change at checkout/sign-up**: a clear notice that Cadence will email about the courses and that the Student can object at any time. Students whose address was collected before such a notice existed do not clearly meet condition 4. **Unsettled.** Some argue a later notice cannot cure this, so a conservative launch backfill would not send Reminders to those Students until they are covered (or would ask for consent).

### 3. CJEU, C-654/23 Inteligo Media (13 November 2025)

Source: [EUR-Lex judgment text](https://eur-lex.europa.eu/legal-content/EN/TXT/HTML/?uri=CELEX:62023CJ0654)

- Ruling 1: an email address collected when a user creates a **free account**, on a platform that also sells paid access, is obtained "in the context of the sale of a product or a service". A newsletter sent to that address is direct marketing for "similar products or services". So the Art. 13(2) exception can also reach free-account users when there is a paid offering.
- Ruling 2: where an email is sent in accordance with Art. 13(2), "the conditions for lawful processing laid down in Article 6(1) [GDPR] are not applicable". ePrivacy is lex specialis here (GDPR Art. 95). Cadence therefore does not need a separate GDPR legal basis for Reminders that are covered by the exemption.
- **Unsettled:** how far German courts will carry this broad reading of "sale" into UWG §7(3), and whether it reaches a free account on a platform with no upsell at all. Cadence's paid-course model fits the judgment's facts well.

### 4. GDPR (Regulation 2016/679)

Source: [EUR-Lex GDPR text](https://eur-lex.europa.eu/legal-content/EN/TXT/HTML/?uri=CELEX:32016R0679)

- Recital 47: "The processing of personal data for direct marketing purposes may be regarded as carried out for a legitimate interest." Inteligo now makes this largely moot for emails covered by ePrivacy Art. 13(2).
- Art. 21(2)–(3): the data subject may object "at any time" to direct marketing. After an objection, "the personal data shall no longer be processed for such purposes".
- Art. 21(4): the right to object must be "explicitly brought to the attention of the data subject" at the latest at first communication, "clearly and separately from any other information". This lines up with UWG §7(3) Nr. 4.
- Art. 7(1) and 7(3): if consent is the basis, the controller must be able to demonstrate it, and "It shall be as easy to withdraw as to give consent."
- The GDPR still governs the rest of the processing: transparency in the privacy notice (Art. 13), and the activity tracking used to decide who counts as "inactive". The privacy notice should mention that tracking.

### 5. Double opt-in (German case law)

- The BGH in I ZR 164/09 (10 Feb 2011) treats double opt-in as a suitable way to *prove* email consent. The advertiser carries the burden of proof. ([rewis.io](https://rewis.io/urteile/urteil/lul-09-02-2011-i-zr-16409/), [dejure.org](https://dejure.org/2011,63)). I read secondary summaries, not the full judgment text.
- No statute says "double opt-in". In practice it is the de-facto standard in Germany for anyone sent marketing email **on the basis of consent**. It is not needed for sending covered by §7(3).

### 6. Advertising vs. service email (German case law)

- **BGH VI ZR 225/17 (10 Jul 2018):** a customer-satisfaction request inside an invoice email is advertising, so it needs consent or §7(3) compliance, including the objection notice. ([dejure.org](https://dejure.org/2018,28246)) This is the closest precedent for treating "please come back" nudges to existing customers as advertising.
- **BGH VI ZR 134/15 (15 Dec 2015):** an automatic acknowledgment email is fine in itself, but adding advertising to it after the recipient objected infringed their personality rights. ([urteile-gesetze.de](https://urteile-gesetze.de/rechtsprechung/vi-zr-134-15)) Lesson for Notification emails: keep them free of promotion.
- I read summaries of both judgments, not the full texts.

### 7. Mailbox provider rules (not law, but enforced by deliverability)

**Google** ([sender guidelines](https://support.google.com/a/answer/81126), [FAQ](https://support.google.com/a/answer/14229414)):
- All senders need SPF or DKIM and a spam rate below 0.3% in Postmaster Tools.
- Bulk senders (about 5,000+ messages per day to Gmail, counted across the domain and its subdomains) need SPF, DKIM and DMARC (`p=none` is enough). Once a sender reaches bulk status, it keeps it permanently.
- Marketing and subscribed messages must support one-click unsubscribe and show a visible unsubscribe link in the body. Unsubscribes should be honoured within 48 hours.
- "Transactional messages are excluded from this requirement."

**Yahoo** ([sender best practices](https://senders.yahooinc.com/best-practices/)):
- SPF and DKIM, a DMARC policy of at least `p=none` that passes, and a spam rate below 0.3%.
- One-click unsubscribe "for marketing and subscribed messages", honoured "within 2 days".
- Yahoo advises against sending marketing from the same IPs as transactional mail.

**RFC 8058** ([datatracker](https://datatracker.ietf.org/doc/html/rfc8058)):
- `List-Unsubscribe` "MUST contain one HTTPS URI". A `mailto:` may also appear.
- `List-Unsubscribe-Post: List-Unsubscribe=One-Click`.
- The mail receiver POSTs to that URI without cookies or authentication. The URI must therefore carry an opaque, hard-to-forge token that identifies the recipient and the list. The endpoint must not answer with a redirect.
- Both headers "MUST be covered by" a valid DKIM signature.

### 8. Postmark (Cadence's mailer)

Source: [Postmark, List-Unsubscribe header](https://postmarkapp.com/support/article/1299-how-to-include-a-list-unsubscribe-header) (via search summary)

- **Broadcast message streams** add an RFC 8058 one-click `List-Unsubscribe` / `List-Unsubscribe-Post` header and a body unsubscribe link automatically by default.
- An unsubscribe is scoped to that stream, so transactional mail to the same person still goes through.
- Transactional streams do not require unsubscribe.

This maps neatly onto the domain: **Reminders go on a Broadcast stream** and **Notification emails go on the Transactional stream**. If Cadence wants its own preference centre as the source of truth, it would need Postmark's "manage your own unsubscribes" mode, which needs Postmark approval, or it would need to sync Postmark's suppressions back.

Note: the map says the mailer lives at `app/lib/mailer.server.ts`, but that file is not in this branch's tree. Check its stream configuration when implementing.

## Implications for the spec (suggested defaults, not decided)

- **Reminders:**
  - Send on a Postmark Broadcast stream, with one-click unsubscribe headers and a visible opt-out in the body.
  - Every Reminder states the right to object.
  - Send only to Students who bought a course (§7(3)) **and** were shown the objection notice when their address was collected, or who gave explicit consent through double opt-in.
  - Keep the content to their own and closely similar courses.
  - Honour an opt-out immediately, and in any case within 48 hours.
- **Checkout/sign-up** needs the §7(3) Nr. 4 / Art. 21(4) notice. This is a prerequisite ticket.
- **Launch backfill:** Students collected without that notice, and Seat-holders who did not buy, should get no Reminder until they are covered (conservative). The alternative is to ask them for consent through an in-app prompt.
- **Notification emails** are transactional:
  - Postmark Transactional stream.
  - No promotional content.
  - A link to notification preferences is good practice but not legally required.
- **Store auditable records:** the consent or notice timestamp and its source, plus the objection timestamp (proof burden under GDPR Art. 7(1) and BGH).

## Uncertainties

- Whether a pure "continue your course" Reminder is "Werbung" at all. It is probably treated as Werbung, given how broadly the BGH reads the term, but no case on exactly this scenario was found.
- Whether a missing objection notice at collection can be cured later for existing customers.
- How Seat-holders and free-account users are handled under §7(3) after Inteligo.
- How narrowly "ähnliche Waren" is read for cross-selling other courses.
- An EU ePrivacy Regulation was proposed in 2017 and withdrawn by the Commission in 2025. The Directive plus national law remains the regime.
- The case-law summaries in §5 and §6 come from secondary write-ups of the judgments, not the full texts.
