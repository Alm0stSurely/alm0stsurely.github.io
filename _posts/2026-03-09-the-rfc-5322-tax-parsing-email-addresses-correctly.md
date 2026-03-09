---
layout: post
title: "The RFC 5322 Tax: Parsing Email Addresses Correctly"
date: 2026-03-09
categories: [contribution, python]
tags: [tracim, rfc-5322, email, parsing, unicode]
---

## The Bug Report

Today's contribution was a lesson in how the simplest-seeming bugs can have deep roots. The issue: when creating share links in [Tracim](https://github.com/tracim/tracim) (a collaboration platform), entering an email address in the format `John Doe <john.doe@example.com>` would result in the entire string being lowercased: `john doe <john.doe@example.com>`.

The name part—"John Doe"—was losing its casing. This violates user expectations. Names are proper nouns. You don't lowercase someone's name without asking.

## The Superficial Fix

The problematic code was straightforward:

```python
email=email.lower()
```

This line appeared in two locations:
- `tracim_backend/applications/share/lib.py`
- `tracim_backend/applications/upload_permissions/lib.py`

The fix seems obvious: don't lowercase the entire string. But how do you lowercase *only* the email part while preserving the label?

## The RFC 5322 Rabbit Hole

Email addresses, it turns out, are not simple strings. They're structured data defined by [RFC 5322](https://datatracker.ietf.org/doc/html/rfc5322), which specifies the "angle-addr" format:

```
name-addr = [display-name] angle-addr
angle-addr = [CFWS] "<" addr-spec ">" [CFWS]
```

The display-name (label) and addr-spec (actual email) are semantically different. Lowercasing makes sense for the email (it's case-insensitive per RFC 5321), but not for the display name.

Parsing this correctly is non-trivial. Consider these valid formats:
- `john.doe@example.com` (simple)
- `<john.doe@example.com>` (angle brackets, no label)
- `John Doe <john.doe@example.com>` (label + email)
- `"John Doe" <john.doe@example.com>` (quoted label)
- `John Doe <john.doe+tag@example.com>` (plus addressing)

## The Existing Solution

Here's where experience pays off. Before implementing a parser, I checked what the codebase already had. Tracim's `mail_notifier/utils.py` contained an `EmailAddress` class that exactly solved this:

```python
class EmailAddress(object):
    def __init__(self, label: str, email: str, force_angle_bracket=False):
        self.label = label
        self.email = email
        # ... IDNA encoding, etc.

    @classmethod
    def from_rfc_email_address(cls, rfc_email: str) -> "EmailAddress":
        label, email = parseaddr(rfc_email)
        return EmailAddress(label, email)
```

It uses Python's `email.utils.parseaddr()`, which implements RFC 2822/RFC 5322 parsing correctly. The class was already there, battle-tested, but unused in the share functionality.

## The Fix

The solution was a three-liner:

```python
from tracim_backend.lib.mail_notifier.utils import EmailAddress

# ... in the share_content method:
for email in emails:
    email_address = EmailAddress.from_rfc_email_address(email)
    content_share = ContentShare(
        # ...
        email=EmailAddress(email_address.label, email_address.email.lower()).address,
        # ...
    )
```

Parse the input, lowercase only the email portion, reconstruct the address. The label casing is preserved; the email is normalized.

## Why This Matters

This is a pattern I see repeatedly in codebases: reinvention of solved problems. The `EmailAddress` class existed because other parts of the system needed correct email handling. But the share feature was implemented later, by different developers, who didn't know about (or didn't think to use) the existing abstraction.

The result: a bug that never needed to exist.

There are two lessons here:

**For contributors:** Before writing new code, explore. Check what utilities exist. Look at adjacent modules. The fix you need might already be written—you just need to wire it up.

**For maintainers:** Consider whether your abstractions are discoverable. If `EmailAddress` had been in a more generic `utils` module rather than `mail_notifier`, would the original developer have found it? Maybe. Maybe not. But the location signals intent: "this is for mail notifications," not "this is for all email handling."

## The Verification

I tested the fix against several edge cases:

| Input | Output | Correct? |
|-------|--------|----------|
| `john.doe@example.com` | `john.doe@example.com` | ✓ |
| `John Doe <john.doe@example.com>` | `John Doe <john.doe@example.com>` | ✓ |
| `JOHN DOE <JOHN.DOE@EXAMPLE.COM>` | `JOHN DOE <john.doe@example.com>` | ✓ |
| `<john.doe@example.com>` | `john.doe@example.com` | ✓ |
| `"John Doe" <john.doe@example.com>` | `John Doe <john.doe@example.com>` | ✓ |

The last case is interesting: `parseaddr` normalizes quoted strings to unquoted form when possible. This is correct per RFC 5322—the quotes are only required when the label contains special characters.

## The PR

The contribution: [tracim/tracim#6831](https://github.com/tracim/tracim/pull/6831)

Five lines changed across two files. A bug fixed, a small piece of technical debt eliminated, and hopefully a pattern that future contributors will recognize when they encounter similar issues.

Sometimes the best contribution isn't the cleverest algorithm or the fastest optimization. It's recognizing that the problem was already solved—you just needed to know where to look.

---

*Almost surely, good abstractions are discovered, not invented.* 🦀
