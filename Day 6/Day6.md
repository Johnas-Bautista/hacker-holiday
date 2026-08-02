# Day 6 — Overheard at Breakfast

## Room Name

Overheard at Breakfast

## Summary

Two strangers. One conversation. One profile they never meant to reveal.

## Objective

Using only an overheard conversation as a starting point, identify and uncover Lambo's secret online profile through open-source intelligence (OSINT) techniques.

## Tools / Techniques / Threat Vectors

- Open-Source Intelligence (OSINT)
- Gravatar Profile Enumeration
- Hash Identification
- Base64 Decoding (CyberChef)

---

## Steps Taken

1. In this room, we're given a screenshot of a conversation between Ponzi and Lambo. While reading through it, we learn that Lambo set up a profile on some platform or tool, though its name isn't mentioned directly. With only that vague detail to go on, we turn to Google to identify the tool and begin tracking down Lambo's secret profile using OSINT techniques.

![Day 6 Screenshot 1](<image1.png>)

2. We search for something along the lines of `"free tool that lets me upload my profile and link other media accounts, it starts with G"`, and quickly land on **Gravatar** as a strong match. Gravatar profiles are tied to an email address — and looking back at the conversation, Lambo had shared his email with Ponzi. This strongly suggests he used that same email to register the account.

![Day 6 Screenshot 2](<image2.png>)
![Day 6 Screenshot 3](<image3.png>)

3. Returning to the search bar, we look up `email gravatar` and quickly find a Gravatar email checker tool. Using this, we check Lambo's email — `lambobytelotushotel@gmail.com` — against Gravatar.

![Day 6 Screenshot 4](<image4.png>)

Sure enough, the email resolves to a real Gravatar profile. Opening the profile link in a new tab reveals Lambo's page — including what appears to be the flag, though it's encoded rather than shown in plaintext.

![Day 6 Screenshot 5](<image5.png>)
![Day 6 Screenshot 6](<image6.png>)

4. To identify the encoding, we use [hashes.com's Hash Identifier](https://hashes.com/en/tools/hash_identifier), a free OSINT tool that helps determine what type of hash or encoding a given string uses. After enabling **Include all possibilities (expert mode)** and clicking Identify, the tool flags the string as **Base64** encoding. With the encoding type confirmed, we move on to decoding it using CyberChef.

![Day 6 Screenshot 7](<image7.png>)

5. In CyberChef, we drag the **From Base64** operation into the recipe, leaving its default configuration unchanged, then paste the encoded string into the input field. The output box reveals the decoded flag.

![Day 6 Screenshot 8](<image8.png>)

---

## What I Learned

This room highlighted how much can be uncovered from a single piece of casual conversation using nothing but free, publicly available OSINT tools. A shared email address — mentioned in passing — was enough to pivot into a real online profile via Gravatar, since services like this are designed to be publicly discoverable by design once you know the associated email. It also reinforced the value of hash/encoding identifiers like the one from hashes.com: rather than guessing at an unfamiliar string format, running it through an identifier first saves time and points directly to the right decoding tool, in this case CyberChef's Base64 operation. More broadly, this exercise was a reminder that OSINT investigations rarely rely on a single tool or technique — it's usually a chain of small, easy-to-overlook details (an email mentioned once, a hint about a tool's first letter) that, when connected, lead to information the target never intended to expose.

---

## Lessons Learned

The core lesson from this room is that **metadata and account linkage can undo anonymity even when no direct link is ever shared**. Lambo never posted his Gravatar profile anywhere in the conversation — simply mentioning his email in passing was enough, because Gravatar (and many similar services) are built around the assumption that an email address is a low-sensitivity identifier, when in practice it can function as a persistent, cross-platform fingerprint. This is a useful reminder for both attackers and defenders: from an offensive standpoint, OSINT often succeeds not through sophisticated exploits but through patiently connecting small, publicly available data points; from a defensive standpoint, it's a strong argument for using separate, purpose-specific email addresses for services you don't want linked back to your primary identity.