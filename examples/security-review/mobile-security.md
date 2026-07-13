# Example: MobileSecurity on ClariNote PRD

> Agent: `MobileSecurity` (Sonnet) · Input: [`../sample-system/PRD.md`](../sample-system/PRD.md)
> Illustrative domain output. The PRD is thin on mobile detail, so several findings are design-level questions the mobile team must answer before build.

---

## Mobile Security Review

### Critical findings
| # | Area | Finding | Scenario | Fix |
|---|---|---|---|---|
| M-001 | Local PHI storage | The React Native clinician app displays and uploads PHI (§4) but says nothing about on-device storage. RN apps commonly cache API responses, images, and drafts to unencrypted `AsyncStorage` / app sandbox, and OCR'd document images may be written to the gallery/temp. | A lost or stolen clinician phone, or a malicious app on a jailbroken device, reads cached PHI from unencrypted local storage or OS backups. | Store any PHI at rest in the Keychain/Keystore-backed encrypted storage; avoid caching PHI where possible; exclude PHI paths from iCloud/Android backups; clear caches on logout. |

### High findings
| # | Area | Finding | Scenario | Fix |
|---|---|---|---|---|
| M-002 | Transport / certificate pinning | No certificate pinning described for the app→API connection carrying PHI and JWTs. | On a hostile network (or with a user-installed proxy CA), an attacker MITMs the connection and captures PHI and the session JWT — which, given the shared-secret model (Platform P-001), is especially valuable. | Implement certificate/public-key pinning with a rotation plan; reject user-added CAs for the API domain. |
| M-003 | Deep links / IPC | React Native apps often register custom URL schemes/universal links. If ClariNote exposes deep links referencing `patient_id`/`summary_id`, another app or a crafted link could trigger unauthenticated navigation or leak identifiers. | A malicious app or phishing link invokes `clarinote://summary/12345`, and the app opens PHI without a fresh auth check, or the identifier leaks via the link. | Require authenticated session for all deep-link targets, validate/authorize the referenced object server-side, and avoid putting PHI/identifiers in link parameters. |

### Medium / Low findings
| # | Area | Finding | Scenario | Fix |
|---|---|---|---|---|
| M-004 | Screen/data leakage | No mention of screenshot protection, screen-recording, or backgrounding blur for PHI screens. | PHI is captured in the app switcher snapshot or via screen recording. | Mark PHI screens as secure (FLAG_SECURE on Android; blur on background for iOS). |
| M-005 | Session handling on device | JWT storage location on device unspecified; if in `AsyncStorage`, it's readable on a compromised device. | Token theft from device storage grants API access. | Store tokens in Keychain/Keystore; short lifetimes; biometric re-auth for sensitive actions. |
| M-006 | Root/jailbreak posture | No integrity/anti-tamper checks described. | A rooted device or repackaged app extracts PHI or pins. | Add root/jailbreak detection as defense-in-depth (not sole control) for a PHI app. |

### What's done well
- Providing a dedicated clinician app (rather than only a mobile web view) gives the team control over storage, pinning, and screen-protection — the mechanisms exist to fix the above at the platform level.

### Verdict
**HIGH RISK (pending detail)** — The PRD doesn't specify on-device storage or transport hardening for an app that carries PHI and session tokens. Treat M-001 (local PHI at rest) and M-002 (pinning) as must-answer before build; a lost phone should never be a PHI breach. Request the mobile design detail and re-review.
