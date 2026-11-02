
# The DoD Mishap: How I Found a Critical Crypto Vulnerability in Military Healthcare Apps and Spent Five Months Being Ignored

---

## Preface

I'm Aiden. I'm 14. I started doing bug bounty and vulnerability disclosure in May 2026 because I thought it'd be fun. It was and is! Until I found something real, and the system that's supposed to handle it fell apart at every step.

This is the full story of how I found a cryptographic vulnerability in a U.S. Department of Defense healthcare authentication library, reported it properly, got dismissed, escalated it to MITRE, got ghosted, escalated it to CISA, got confirmed — and still ended up with nothing.

Every date, every message, every response (and non-response) is real. I'm publishing this because CISA explicitly told me I'm free to share my findings, and because I think people need to see what independent vulnerability reporting actually looks like in 2026.


I might be writing this with pure rage
---

## The Finding

In June 2026, I was looking at `mobile.health.mil`, the U.S. Department of Defense's mobile health platform. Specifically, the MWIST (Military WIC Screening Tool) progressive web app.

The first thing I noticed was that JavaScript source maps were publicly accessible. The main source map was 6.5MB and contained 225 unminified source files — application components, authentication logic, route guards, eligibility screening algorithms, everything. That shouldn't be in production.

But the source maps were just the entry point. What they revealed was worse.

The MWIST app uses an npm package called `dha-encryption` (via `dha-login`) for client-side credential encryption. When I looked at how it worked, I found three cryptographic weaknesses that, combined, completely break the security model:

### 1. Hardcoded Default Salt

The `getKeyFromPassword()` function uses `"h0n3yp0t!@"` as the default PBKDF2 salt. If developers don't override the `seed` parameter — and the production apps don't — every deployment shares the same salt.

```
async function getKeyFromPassword(t, e = "h0n3yp0t!@") {
```

You can confirm this yourself against the live production bundle:

```bash
curl -sk "https://mobile.health.mil/mobile_apps/mwist/assets/index-CEo-9ZJ1.js" | grep -oP '.{0,30}h0n3yp0t.{0,30}'
```

### 2. 1,337 PBKDF2 Iterations

The library uses 1,337 iterations of PBKDF2-SHA256. OWASP recommends a minimum of 600,000. That's 448x below standard. When the password is a 4-digit PIN, this makes brute-force trivial.

### 3. Deterministic IV Derivation

The AES-256-GCM key and IV are both derived from the same PBKDF2 output:

```
PBKDF2(password, salt, 1337, 384 bits) → first 256 bits = AES key, last 128 bits = IV
```

Same password + same salt = same IV, every time. Reusing a nonce with AES-GCM is a known catastrophic failure — it destroys the authentication tag guarantee and enables plaintext recovery.

### The Impact

An attacker with access to a user's encrypted data (physical access to the device, XSS, malware, backup extraction) can:

1. Brute-force the entire 4-digit PIN keyspace in **13 seconds** on a regular laptop
2. Decrypt stored credentials and security question answers
3. Use the recovered answers to reset the PIN and take over the session

No server interaction required. Entirely offline.

And this isn't limited to one app. I confirmed the same vulnerable auth flow in at least four DHA progressive web apps:

- **MWIST** — `mobile.health.mil/mobile_apps/mwist/`
- **DRES** — `mobile.health.mil/mobile_apps/dres/`
- **AFMSSLW** — `mobile.health.mil/mobile_apps/afmsslw/`
- **Biodefense** — `mobile.health.mil/mobile_apps/biodefense/`

All four contain the same hardcoded salt, the same 1,337 iterations, and the same deterministic IV derivation. The `dha-encryption` package is the common dependency, and the latest published version (v3.3.1, March 2024) still has all three weaknesses.

These apps handle sensitive data for military families — WIC eligibility screening, which includes income, rank, family size, and pregnancy status.

I built a full proof-of-concept that demonstrates all three weaknesses using only the public npm package: [github.com/AidenMohan/dha-encryption-3.3.1-poc](https://github.com/AidenMohan/dha-encryption-3.3.1-poc)

---

## The Report — and the Response

### June 16, 2026: HackerOne Report to DoD VDP

I submitted a detailed report to the U.S. Department of Defense Vulnerability Disclosure Program through HackerOne (Report #3804724). The report included:

- Full technical breakdown of all three cryptographic weaknesses
- Proof that the hardcoded salt and iteration count are in the live production JavaScript bundles
- A working brute-force PoC that cracks the PIN space in 13 seconds
- The attack chain from encrypted blob to full session takeover
- OWASP and NIST references

Everything was verified against publicly accessible resources. No DoD systems were accessed with credentials. No authenticated access was used.

### June 17: "Can you show proof of exploitation?"

The triager responded with a request for additional information:

> "Can you please answer the following questions? Are you able to show proof that this can be used for exploitation or gain access to sensitive information?"

The catch: to demonstrate the full chain on the live system, I would need an active account on MWIST or DRES. These apps require DoD credentials I don't have. Accessing another user's data would be unauthorized. I explained this and offered to demonstrate with a test account if they could provide one.

### June 18: "We do not take brute forcing issues"

The triager came back with:

> "We do not take brute forcing issues, can you confirm that this domain has a login portal?"

This is where the disconnect became clear. The triager was treating this as a brute-force login attack against a web form. It's not. It's a **cryptographic failure** — the attack happens entirely offline against encrypted data that's already been extracted. There is no login portal involved. The attacker never contacts the server.

I explained this again, with the curl command showing the hardcoded salt in the production JavaScript, and the distinction between "brute-forcing a login" and "brute-forcing weak encryption offline."

### June 22: Closed as "Informative"

After I provided additional confirmation that the same vulnerability existed across four separate DHA applications (MWIST, DRES, AFMSSLW, and Biodefense), the triager closed the report:

> "Proof needs to be provided to confirm vulnerability, no proof is provided that the attacker is able to gather health data or use the salt/brute forced pin code to gain sensitive information, closing as informative."

The vulnerability was confirmed in the production source code. The PoC demonstrated full PIN recovery in 13 seconds. The hardcoded salt was visible in the live JavaScript bundle. But because I couldn't — without breaking the law — log into a military healthcare application and extract my own encrypted data to brute-force, the report was dismissed.

---

## The Escalation: MITRE

### June 26: CVE Request

With the DoD VDP closed, I submitted a CVE request to MITRE through their CNA of Last Resort (CNA-LR) process. This is the escalation path for when a vendor doesn't engage.

MITRE reserved a CVE ID (CAN-2026-2032508) and created a full record:

- CWE-916: Use of Password Hash With Insufficient Computational Effort
- CVSS 3.1 Base Score: **7.7 (HIGH)**
- Attack Vector: Local
- Confidentiality Impact: High
- Integrity Impact: High
- Status: **Published**
- Scope: **IN-CNA-LR-SCOPE** (confirmed within their scope)

The record was complete. Description, references, metrics, credits, everything.

### September 7: Follow-up

After hearing nothing for over two months, I followed up:

> "Hiya, did this need anymore info or anything? Any chance I can get an eta on when this will be reviewed? Thanks!"

No response.

### October 2: Silent Denial

What I didn't know at the time — and only discovered later by digging through MITRE's public CVE request tooling — was that on October 2, my CVE request was silently denied. The `x_Denied` flag was set to `true` in the record. Nobody told me. (even though they were meant to email on any updates)

The record that MITRE themselves had created — with the CVSS score, the CWE classification, the in-scope confirmation — was rejected without any communication to me.

I found this out by looking at my own request data at [mitre.github.io/mitre-cve-roles/cve-id-request](https://mitre.github.io/mitre-cve-roles/cve-id-request/#source). The JSON shows everything: the reservation, the full record, the follow-up I sent that was never answered, and the silent denial.

---

## The Escalation: CERT/CC and CISA

### September 8: CERT/CC Report

While waiting for MITRE (who had already silently denied me, though I didn't know), I escalated through CERT/CC. CISA's Vulnerability Response and Coordination team received my report and opened case **VU#928288**.

### September 15: CISA Confirms the Findings

Eoin from CISA responded:

> "I checked out your report and all three issues you describe seem legitimate, and I appreciate that you reached out to the DoD/DHA and MITRE already! I will flag this with my team and see if we can easily get in touch with the right person to address your findings."

For the first time in three months, someone with authority looked at the work and said it was real.

The expected public disclosure date was set to **October 26, 2026**.

### October 17: Check-in

I followed up asking if everything was on track for the disclosure date.

No response.

### October 26: Disclosure Date

The disclosure date arrived. I messaged asking if it was ready to be publicly shared, and that I wouldn't talk about it until I heard back.

No response.

### November 2: "Does not warrant CVE assignment"

A week past the disclosure date, Eoin responded:

> "Thanks for checking in and apologies for the delay here, we have quite a significant volume of cases right now. I did get in touch with the MITRE CNA, and after a little discussion we both agreed that these issues do not warrant CVE assignment. Thank you for the report, and you are welcome to publish your own findings as you see fit!"

After five months:

- DoD VDP closed it as "Informative"
- MITRE reserved a CVE, scored it HIGH, confirmed it in-scope, silently denied it, and never told me
- CISA confirmed the findings were legitimate, set a disclosure date, missed it, and then agreed with MITRE that it doesn't warrant a CVE

No explanation for why it doesn't warrant assignment. No technical reasoning. Just "does not warrant."

---

## The Context: Why This Happened

I was angry when I got that response. I still am. But I think it's important to understand what's happening around this.

The CVE program nearly shut down entirely in April 2025 when CISA's contract with MITRE almost expired. It was saved with a last-minute 11-month extension — the night before it would have gone dark.

MITRE laid off over 400 employees after the Trump administration cancelled $28 million in contracts. CISA's workforce dropped from roughly 3,400 to 2,400 — losing a third of its staff. The proposed FY2027 budget cuts CISA by another $700 million.

When Eoin told me "we have quite a significant volume of cases right now," It was an understatement. The agencies responsible for coordinating vulnerability disclosure in the United States are being gutted while the number of vulnerabilities keeps growing.

That doesn't make the outcome acceptable. But it does make it systemic rather than personal.

---

## The Vulnerability Is Still There

As of the date of this publication, the `dha-encryption` npm package (v3.3.1) still contains:

- The hardcoded default salt `"h0n3yp0t!@"`
- 1,337 PBKDF2 iterations (448x below OWASP minimum)
- Deterministic IV derivation that breaks AES-GCM

The production JavaScript bundles on `mobile.health.mil` still contain this code. The PoC still works.

The bundle hash has changed since my original report — from `index-CEo-9ZJ1.js` to `index-Dad7YiSA.js` — meaning a new build was deployed. But the vulnerable code is identical. You can verify right now:

```bash
curl -sk "https://mobile.health.mil/mobile_apps/mwist/assets/index-Dad7YiSA.js" | grep -oP '.{0,30}h0n3yp0t.{0,30}'
# Returns: n:Iq});async function Bq(t,e="h0n3yp0t!@"){const n=new TextEncoder,r

curl -sk "https://mobile.health.mil/mobile_apps/mwist/assets/index-Dad7YiSA.js" | grep -oP '.{0,30}1337.{0,30}'
# Returns: ",salt:n.encode(e),iterations:1337},a=await crypto.subtle.derive
```

No CVE was assigned. No advisory was published. No patch has been released. They shipped a new build and didn't fix it.

---


## Timeline

| Date | Event |
|------|-------|
| June 16, 2026 | Report submitted to DoD VDP via HackerOne (#3804724) |
| June 17, 2026 | Triager requests proof of exploitation |
| June 18, 2026 | Triager: "We do not take brute forcing issues" |
| June 22, 2026 | Additional affected apps confirmed (MWIST, DRES, AFMSSLW, Biodefense) |
| June 22, 2026 | Report closed as "Informative" |
| June 26, 2026 | CVE requested from MITRE CNA-LR |
| June 26, 2026 | CVE reserved (CAN-2026-2032508), CVSS 7.7 HIGH, IN-CNA-LR-SCOPE |
| September 7, 2026 | Follow-up to MITRE — no response |
| September 8, 2026 | Report submitted to CERT/CC |
| September 15, 2026 | CISA confirms findings are legitimate (VU#928288) |
| October 2, 2026 | MITRE silently denies CVE — no notification sent |
| October 26, 2026 | CISA disclosure date passes — no response |
| November 2, 2026 | CISA: "Does not warrant CVE assignment" |
| November 2026 | This writeup published. Vulnerability remains unpatched. |

---

## References

- [dha-encryption on npm](https://www.npmjs.com/package/dha-encryption)
- [Proof of Concept](https://github.com/AidenMohan/dha-encryption-3.3.1-poc)
- [OWASP PBKDF2 Recommendations](https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html#pbkdf2)
- [NIST SP 800-38D — AES-GCM](https://nvlpubs.nist.gov/nistpubs/Legacy/SP/nistspecialpublication800-38d.pdf)

---

*If you run a project and someone reports a vulnerability, please listen. It doesn't matter who they are. It matters what they found.*
