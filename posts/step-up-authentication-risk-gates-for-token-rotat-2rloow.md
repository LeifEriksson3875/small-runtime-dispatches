# Step-Up Authentication: Risk Gates for Token Rotation and Stolen Sessions

Require step-up authentication immediately before a high-impact action when the existing session no longer provides enough confidence for that action. **TL;DR: keep ordinary work fast, but demand a fresh, stronger proof before rotating recovery credentials, changing account control, or responding to evidence that a session was stolen.** The deciding constraint is abuse resistance, not how recently the user opened the app.

For a developer tool, token rotation is the useful test case. A valid browser session may be sufficient to read a project, yet insufficient to mint a new refresh token after an IP jump, a suspicious automation burst, or a recovery event. Step-up verification raises assurance for that one transition. It should not silently turn every page load into another login.

## What does step-up authentication actually change?

Step-up authentication changes the proof required for a sensitive operation. The application evaluates the current session, the requested action, and relevant risk signals, then asks for an additional or stronger authenticator when the present assurance is too low. OWASP recommends reauthentication after high-risk events and for critical actions; examples include password changes, recovery, and suspicious account activity.

The distinction from a blanket multi-factor login matters. A service can require two factors at every sign-in without making an action-aware decision. Conversely, a session that began with a strong authenticator may still need fresh user presence before a destructive or account-controlling action. NIST describes authentication intent as demonstrating that a claimant intended to authenticate, which helps resist malware-driven use of an endpoint when the user did not mean to approve it.

Freshness is part of the decision, but it is not the whole decision. A five-minute-old session may be hostile if its cookie was copied. A six-hour-old session may be acceptable for reading public build logs. The policy needs both an assurance requirement and a maximum age for the proof attached to the sensitive action.

Time alone lies.

This is the practical model:

| Requested action | Existing session | Additional signal | Decision |
| --- | --- | --- | --- |
| Read a repository index | Valid, normal behavior | None | Continue |
| Rotate a refresh token | Valid, recent strong proof | No replay or abuse signal | Continue and rotate |
| Rotate a refresh token | Valid, proof too old | None | Require step-up |
| Revoke all sessions | Valid | Any meaningful uncertainty | Require step-up, then revoke |
| Use an already-rotated token | Any | Token-family replay | Revoke the token family |

The last row is deliberately different. Step-up is not a cure for confirmed replay. Once the server sees an invalidated refresh token reused, asking the same browser for another factor can leave an attacker and the legitimate user racing each other. Revoke the affected family, record the event, and force a clean authentication path.

This pattern has limitations. It cannot repair a compromised endpoint, prove that every request from a verified browser is benign, or make a weak recovery method stronger. It also adds state, transaction contention, support work, and a failure dependency directly in front of sensitive operations. For a low-risk internal tool with short server-side sessions and no durable user credentials, that trade-off may be unjustified; a fresh full login before the rare administrative action can be the smaller design. For a developer platform that issues long-lived refresh tokens, the extra machinery earns its place because a stolen session can otherwise mint a successor credential and preserve access after the original session is noticed. The decision should follow credential lifetime and action impact, not a desire to add another security feature.

## Put the gate at the irreversible transition

The tempting implementation is a middleware rule such as “challenge after 30 minutes.” It is simple, but it binds friction to a clock rather than harm. It also creates a predictable cadence for bots: wait out the challenge, then automate everything available inside the window. Consider the failure sequence. A bot obtains a session cookie, waits until the application presents its routine challenge, relays or completes that challenge, and then uses the resulting broad window to rotate a refresh token, remove a recovery method, and revoke the legitimate browser. The server sees a recent proof at each step because the middleware made freshness global. An action-bound grant changes the sequence: proof for rotation authorizes rotation once, while removing a recovery method requires a separate decision. This costs another verification in the worst case, so combine adjacent actions only when they truly belong to one user intent and one atomic account change.

Put the gate in the command that commits the sensitive state change. For token rotation, that means after authorization and risk evaluation but before the old token is invalidated and the successor is issued. For global session revocation, it means before incrementing the account session epoch or deleting the server-side session set. A UI-only prompt is not a control; an attacker can call the underlying operation directly.

Three values should travel together through that command: the session identifier, the verified account identifier, and evidence of the latest acceptable authentication event. Keep the evidence server-verifiable and narrowly scoped. A boolean named `mfaPassed` is too vague because it says nothing about when the proof occurred, what action it authorizes, or whether it has already been consumed.

One-time authorization is the safer shape. After successful verification, issue a short-lived step-up grant bound to the account, session, action, and a unique identifier. Consume it atomically when the action commits. Binding the grant prevents a proof obtained for exporting logs from being replayed to rotate credentials.

Short-lived means a policy choice, not a universal number. Set it from the time a human needs to finish the exact flow, then measure challenge completion and replay attempts. Longer windows reduce prompts but enlarge the period in which stolen proof remains useful. I would start narrow for credential rotation because the action is short and its blast radius is large, then adjust from observed completion data rather than intuition.

## A focused state transition

The following TypeScript keeps storage and cryptography behind interfaces so the security contract is visible. The important part is the transaction: consume the step-up grant, mark the presented refresh token used, and create its successor as one commit.

```ts
type RotateRequest = {
  accountId: string;
  sessionId: string;
  presentedTokenHash: string;
  stepUpGrant: string;
};

type GrantClaims = {
  accountId: string;
  sessionId: string;
  action: "refresh-token:rotate";
  grantId: string;
  expiresAt: number;
};

interface TokenRecord {
  id: string;
  familyId: string;
  accountId: string;
  sessionId: string;
  usedAt: Date | null;
  revokedAt: Date | null;
}

interface RotationStore {
  transaction<T>(work: (tx: RotationStore) => Promise<T>): Promise<T>;
  findTokenForUpdate(hash: string): Promise<TokenRecord | null>;
  consumeGrant(grantId: string, now: Date): Promise<boolean>;
  markTokenUsed(tokenId: string, now: Date): Promise<void>;
  createSuccessor(parent: TokenRecord, now: Date): Promise<{ rawToken: string }>;
  revokeFamily(familyId: string, now: Date): Promise<void>;
}

declare function verifyGrant(encoded: string): Promise<GrantClaims>;

async function rotateRefreshToken(
  request: RotateRequest,
  store: RotationStore,
  now = new Date(),
): Promise<{ refreshToken: string }> {
  const grant = await verifyGrant(request.stepUpGrant);

  if (
    grant.accountId !== request.accountId ||
    grant.sessionId !== request.sessionId ||
    grant.action !== "refresh-token:rotate" ||
    grant.expiresAt <= now.getTime()
  ) {
    throw new Error("STEP_UP_REQUIRED");
  }

  return store.transaction(async (tx) => {
    const current = await tx.findTokenForUpdate(request.presentedTokenHash);
    if (!current || current.accountId !== request.accountId) {
      throw new Error("INVALID_REFRESH_TOKEN");
    }

    if (current.revokedAt || current.usedAt) {
      await tx.revokeFamily(current.familyId, now);
      throw new Error("REFRESH_TOKEN_REUSE");
    }

    const consumed = await tx.consumeGrant(grant.grantId, now);
    if (!consumed) throw new Error("STEP_UP_REQUIRED");

    await tx.markTokenUsed(current.id, now);
    const successor = await tx.createSuccessor(current, now);
    return { refreshToken: successor.rawToken };
  });
}
```

There is a sharp edge here: returning a generic error is useful at the public boundary, but operations still need distinct internal outcomes. `STEP_UP_REQUIRED` starts a verification ceremony. `REFRESH_TOKEN_REUSE` is a security event and closes the family. `INVALID_REFRESH_TOKEN` should not disclose whether an account or token exists. Map those outcomes to stable client behavior without leaking the detection rule.

The transaction also prevents two concurrent requests from both treating the same refresh token as unused. Database row locking or an equivalent compare-and-set is required; an in-process check followed by a later write has a race. OAuth 2.0 Security Best Current Practice describes refresh token rotation as issuing a new refresh token with every response and invalidating the previous one, while retaining the relationship so replay can be detected.

## Challenge design is an abuse-control decision

A challenge that stops a careful human but yields to automation is negative value. Rate-limit attempts by more than one key: account alone enables denial of service, while IP alone is weak behind shared networks and distributed bot traffic. Combine coarse network signals, account and session history, challenge failures, and the sensitivity of the action. Avoid treating any single heuristic as identity proof.

Bots adapt.

The authenticator choice affects phishing resistance. NIST states that manually entered one-time passwords are not phishing-resistant because the verifier output can be relayed. WebAuthn uses public-key credentials scoped to a relying party, and its user-verification ceremony can provide stronger evidence for a sensitive transition. Recovery codes and support recovery still need a path, but they should not inherit the same confidence merely because they eventually produce a valid session.

Fail closed for credential rotation when verification infrastructure is unavailable. Reading already-authorized, low-risk data may follow a different availability policy. This split is uncomfortable during an outage, yet issuing durable credentials without the required proof converts an availability problem into account compromise.

That is an explicit availability trade-off.

Do not expose why a challenge appeared in enough detail to teach an attacker the threshold. The user needs a clear action and a recovery route. The telemetry needs the full reason code.

## Revocation must survive the stolen session

After a user reports a stolen session, verifying them and deleting only the current cookie is incomplete. The system must invalidate the server-side session and associated refresh-token family. If the account action is “sign out everywhere,” revoke every eligible session under the account after fresh verification, with deliberate exceptions only for separately modeled machine credentials.

Keep browser sessions and automation credentials distinct. Developer tools often have headless jobs that cannot answer an interactive challenge. Do not waive step-up for them and call the result secure. Give workload credentials explicit scopes, rotation rules, and revocation paths; prevent them from invoking human account-control operations in the first place.

OWASP also recommends rotating tokens after reauthentication and invalidating sessions after reauthentication. That ordering prevents a newly verified user from continuing on the same potentially exposed session identifier. Preserve enough linkage in the audit trail to explain which session was replaced without logging raw credentials or challenge secrets.

Recovery deserves the harshest review. An attacker who controls email may be able to pass an email challenge, so the recovery path should not automatically authorize removal of a stronger enrolled authenticator. Depending on the account model, use a delay, an existing trusted authenticator, administrator review, or a clearly communicated lockout period. Each adds support cost. The alternative is making the weakest recovery factor the real security boundary.

## Measure before copying this policy

Ship the policy behind observable reason codes, not a single `challenged=true` counter. Measure challenge rate per sensitive action, completion rate, time to completion, failure and lockout rate, grant replay, refresh-token reuse, and successful revocation latency. Segment operationally useful signals without collecting raw authenticator output or refresh tokens.

Watch cost in two places. Interactive verification has direct infrastructure and support cost; false positives interrupt real work. Under-challenging has a quieter cost in longer attacker dwell time and broader credential issuance. The useful tuning question is not “How do we minimize prompts?” It is “Which signals predict enough harm to justify friction at this transition?”

Test the ugly paths: two concurrent rotations, a consumed grant submitted again, a grant used for the wrong action, a token replayed after its successor is issued, revocation racing with rotation, and verification becoming unavailable between challenge and commit. Include clock skew tests, but keep expiry enforcement on a trusted server clock. Never let the client choose the authentication time.

The final rule is compact: **step up before an account-controlling transition when current assurance or freshness is insufficient; revoke rather than challenge when replay already indicates compromise.** Copy the architecture only after the measurements show that the gate catches meaningful risk without turning normal developer work into repeated recovery exercises.

## References

- https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
- https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html
- https://www.rfc-editor.org/rfc/rfc9700.html
- https://pages.nist.gov/800-63-4/sp800-63b.html
- https://www.w3.org/TR/webauthn-3/
