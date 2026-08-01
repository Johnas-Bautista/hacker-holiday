# Day 3

## Room Name

Complimentary

## Summary

Install the free app and it hands your phone a set of cloud keys, the same set it hands everyone. They're read-only, but read-only of every guest's contacts, location, and passwords, not just Lambo's. She gave consent. Technically.

## Objective

Find out how the app knows anything about you at all, and see what else it's willing to hand over.

## Tools/Technique/Threat Vector

AWS IAM Misconfiguration

## Steps Taken

1. As usual we inspect elements inside the browser, we also check the network tab to see some request made by the client's browser. Some of the request made includes an **/app.js**
![Day 3 Screenshot 1](<image1.png>)

The app has no login screen, so all its logic — including how it talks to AWS — has to live in the client-side JavaScript. `/app.js` is the bundle the browser loads to render the dashboard, which makes it the first place to look for how the app "just knows" who you are without asking you to sign in.

2. Now we check the contents of **/app.js** for some clues. And look what we have here an AWS credentials
![Day 3 Screenshot 1](<image2.png>)

Inside the bundle we find three hardcoded values: an `IDENTITY_POOL_ID`, the `AWS_REGION`, and a `TABLE_NAME`. This isn't a real access key/secret pair sitting in the file — it's a **Cognito Identity Pool ID**, which is effectively a public "front door" that the browser uses to request temporary AWS credentials on the fly. Because the pool has unauthenticated identities enabled, anyone who loads the page can walk up to this door and get issued working AWS credentials with zero login.

3. What do we do here, we are going to try
```text
    aws cognito-identity get-id \
    --identity-pool-id "us-east-1:836c0949-292d-485b-b532-52d5ca7bb688" \
    --region us-east-1
```
![Day 3 Screenshot 1](<image3.png>)
This returns something like `{"IdentityId": "us-east-1:xxxxxxxx-..."}`.

Using the pool ID pulled from `app.js`, we ask Cognito to register us as a guest. Since the pool allows unauthenticated access, Cognito doesn't ask for a password or token — it just assigns a random `IdentityId` and treats us as a valid, if anonymous, user of the app.

4.
```text
aws cognito-identity get-credentials-for-identity \
  --identity-id "<IdentityId from step 1>" \
  --region us-east-1
```
![Day 3 Screenshot 1](<image4.png>)
This returns a Credentials block with `AccessKeyId`, `SecretKey`, and `SessionToken`.

With that `IdentityId` in hand, we exchange it for real, temporary AWS credentials. This is the actual mechanism the challenge is pointing at: Cognito hands out a short-lived `AccessKeyId`/`SecretKey`/`SessionToken` tied to whatever IAM role the pool's "unauthenticated role" is configured with — this is exactly what the app's own JavaScript does silently in the background every time someone opens it, which is why it never needs a login screen.

5. Now copy paste the output into this command
```text
export AWS_ACCESS_KEY_ID="<AccessKeyId>"
export AWS_SECRET_ACCESS_KEY="<SecretKey>"
export AWS_SESSION_TOKEN="<SessionToken>"
```
![Day 3 Screenshot 1](<image5.png>)

These `export` commands load the temporary credentials into the current shell session so that any subsequent `aws` command automatically authenticates using them, instead of whatever credentials (or none) were configured before.

then run this command to confirm what identity you're operating as
```text
aws sts get-caller-identity
```
![Day 3 Screenshot 1](<image6.png>)
That confirms it: you're authenticated as `complimentary-cognito-unauth-role` — the IAM role Cognito hands to anyone who shows up with no login.

`get-caller-identity` is a sanity check: it proves the exported credentials are live and shows exactly which IAM role we're now assuming. Seeing `complimentary-cognito-unauth-role` in the returned ARN confirms these aren't dummy values — they're a genuine assumed-role session with whatever permissions that role has been granted.

6. Now try the dynamoDB scan using the table name and region we saw at the **/app.js** file

```text
aws dynamodb scan \
  --table-name complimentary-GuestWellnessProfiles \
  --region us-east-1
```

![Day 3 Screenshot 1](<image7.png>)

after gradually revealing the display we can see the flag value inside a column of a table
![Day 3 Screenshot 1](<image8.png>)

---
The app's intended flow only ever fetches *your own* record from this table. `scan`, however, ignores that intent entirely and requests every item in the table at once. If the IAM policy attached to `complimentary-cognito-unauth-role` had scoped DynamoDB access to each caller's own partition key (e.g. via a `dynamodb:LeadingKeys` condition tied to the Cognito identity), this call would have been denied. Because that restriction was missing, the scan returned every guest's profile — including one containing the flag — proving that access control was enforced only by the frontend UI, not by AWS itself.

---

## What I Learned

**"No login" doesn't mean "no credentials" — it means the credentials are issued silently.**
Apps that skip a sign-in screen still need some way to know what you're allowed to see. In this case that "something" was an AWS Cognito Identity Pool with unauthenticated identities enabled, sitting quietly in the client-side JS (`app.js`). The moment the page loads, the browser calls Cognito behind the scenes, gets back a real, working AWS `AccessKeyId`/`SecretKey`/`SessionToken`, and uses those to talk to AWS directly — no backend API required. The friction-free UX was the vulnerability's delivery mechanism.

**The client-side bundle is often the whole attack surface.**
Everything needed to reconstruct the app's AWS access — identity pool ID, region, and DynamoDB table name — was sitting in plaintext inside `app.js`. Anything shipped to the browser has to be treated as public; there's no such thing as a "hidden" config value in client-side code.

**IAM is the real access-control boundary, not the frontend.**
The app's UI only ever asked for *my* record. But permissions live in the IAM policy attached to the role Cognito hands out (`complimentary-cognito-unauth-role`), not in what the frontend chooses to request. `dynamodb:Scan` ignores any notion of "my data" — it just returns whatever the underlying policy allows. Because that policy didn't include a `dynamodb:LeadingKeys` condition tied to the caller's Cognito identity (`cognito-identity.amazonaws.com:sub`), *every* guest's profile was readable, not just my own.

**Trusting the client is the root cause, every time.**
The gap wasn't a bug in a single API call — it was an assumption that the app's JavaScript would always behave and only ever query its own row. A well-scoped IAM condition would have made that assumption irrelevant, because AWS itself would have refused the broader request regardless of what the client asked for.

**Practical takeaway for defense:**
Whenever an app grants direct client-side access to AWS resources (via Cognito or similar), the IAM policy — not the frontend — must enforce row-level isolation. A `LeadingKeys` condition scoping DynamoDB access to `${cognito-identity.amazonaws.com:sub}` is the specific fix that would have prevented this entire chain from working.
