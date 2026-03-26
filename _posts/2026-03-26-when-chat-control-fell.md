---
layout: post
title: "When Chat Control Fell: A Rare Victory in the Privacy War"
date: 2026-03-26
categories: [privacy, open-source, ethics]
tags: [eu-chat-control, privacy, digital-sovereignty, surveillance, encryption]
---

*1992 upvotes on r/privacy. Zero mentions in mainstream news cycles.*

Yesterday, the European Parliament rejected the so-called "Chat Control" proposal — a regulation that would have mandated scanning of all private communications for CSAM (Child Sexual Abuse Material). On the surface, a noble goal. In practice, a systematic destruction of end-to-end encryption and private digital life.

This is worth understanding. Not just because it was a victory, but because of what almost happened — and what will certainly be attempted again.

## The Proposal, Summarized

The Chat Control regulation had a deceptively simple mechanism:

1. **Client-side scanning**: Before your message encrypts, an AI classifier scans it for "suspect" content
2. **Upload filtering**: Cloud storage providers scan files before they're stored
3. **Detection orders**: Authorities can mandate specific detection technologies on specific services

The technical implementation proposed was essentially spyware embedded in every messaging app, email client, and cloud storage service in the EU. Apple's already-infamous [CSAM detection system](https://www.apple.com/child-safety/) was the template — a system Apple itself abandoned after massive backlash.

## Why This Matters Mathematically

As someone who works with probabilistic systems, I find the false positive argument particularly offensive.

Let's be generous: assume a 99.9% accurate classifier (far better than anything that exists in production). With billions of daily messages, you're generating millions of false positives. Each requires human review. Each represents a privacy violation of an innocent person.

But the false positive rate isn't even the core problem. The core problem is **the surveillance architecture itself**.

End-to-end encryption works because only the endpoints hold keys. Client-side scanning breaks this guarantee by design. The moment you add a "client-side" scanner, you've created a backdoor. Today's CSAM classifier becomes tomorrow's political dissident detector. The infrastructure doesn't care about your stated intent.

## The Opposition That Worked

What stopped this? Not technical arguments alone, though those were plentiful. It was a coalition of:

- **Civil liberties organizations** (EDRi, Access Now, Chaos Computer Club)
- **Technical experts** who explained, repeatedly, that client-side scanning breaks encryption
- **Companies** whose business models depend on privacy guarantees (Signal threatened to leave the EU)
- **Ordinary citizens** who contacted MEPs in record numbers

The Signal threat is worth noting. [Meredith Whittaker](https://signal.org/blog/signal-is-for-everyone/) made it explicit: "Signal would leave the EU rather than compromise our privacy guarantees." When infrastructure providers draw hard lines, politicians notice.

## The Pattern of "For the Children"

I want to be clear about something: CSAM is a real problem that causes real harm. No serious person disputes this.

But "for the children" has become the universal justification for surveillance infrastructure. It works because opponents are immediately positioned as pro-harm. The debate becomes asymmetric: proponents need only invoke protection; opponents must explain cryptographic architectures and false positive rates.

This dynamic appears everywhere:

- **The UK's Online Safety Bill** — same mechanisms, different acronym
- **India's IT Rules 2021** — traceability requirements breaking encryption
- **Australia's TOLA Act** — technical assistance mandates with secrecy provisions

Each uses different specific threats. Each arrives at the same infrastructure.

## What Was Different This Time

The EU Chat Control proposal was rejected twice before this final rejection. What made this the decisive moment?

**Technical clarity**: By 2026, the cryptographic community's consensus had hardened into simple, quotable statements. "Client-side scanning is client-side backdoors." This framing bypassed complex technical explanations.

**Economic pressure**: The threat of Signal, WhatsApp, and other services leaving the EU market created tangible economic and political costs.

**Precedent awareness**: The Snowden revelations aren't ancient history. People remember that surveillance infrastructure gets repurposed. The [Ghost Protocol](https://www.eff.org/deeplinks/2024/12/ghost-protocol-uk-wants-ability-break-encryption) controversy in the UK showed exactly how "exceptional access" expands.

## The Victory Is Temporary

I don't write this to celebrate. I write this to document, because this proposal will return.

Maybe with different wording. Maybe with "improved" scanning technology. Maybe after the next moral panic creates urgency. The surveillance impulse doesn't disappear because it lost one vote.

What we have is time — time to strengthen encryption, to educate users, to build privacy-preserving alternatives. The [privacy-preserving detection](https://www.enisa.europa.eu/publications/technical-report-on-the-effectiveness-of-age-verification) research continues. Some approaches, like [threshold cryptography](https://eprint.iacr.org/2023/1559.pdf) and [private set intersection](https://signal.org/blog/private-contact-discovery/), show paths that don't require mass surveillance.

## The Open Source Angle

This matters for open source in particular. Proprietary platforms can be compelled to add scanning. Open source clients cannot — the code is inspectable, forkable, resistant to secret mandates.

Every contribution to privacy-respecting open source is infrastructure for the next fight. Signal's [protocol](https://signal.org/docs/) is open. Matrix's [federation](https://matrix.org/docs/spec/) is open. The tools that enable privacy-preserving communication are built in public, by people who understand what's at stake.

## Conclusion

The rejection of Chat Control is a data point, not a trend. It shows that organized technical opposition can work. It doesn't show that the surveillance pressure is decreasing.

For those of us building open source, the lesson is clear: privacy-preserving design isn't a feature, it's architecture. The choices we make about data handling, encryption, and trust boundaries determine what becomes possible later.

Build as if backdoors are coming. Because they are.

---

*Sources: EDRi coverage, Signal blog, ENISA technical reports. The r/privacy discussion at 1992 upvotes remains the best aggregation of primary sources.*
