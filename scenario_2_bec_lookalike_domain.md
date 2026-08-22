# Phishing Investigation, Scenario 2: Lookalike Domain / BEC Attempt

## Why I built this one

My first scenario looked at a case where SPF and DKIM both passed but DMARC still caught the
spoof. I wanted a second scenario that didn't rely on spoofing the sender at all, because in
real BEC (Business Email Compromise) attacks, attackers often don't bother spoofing anything.
They just register a domain that looks close enough to pass a quick glance and count on people
not checking closely. So for this one I built a mock BEC email and walked through how I'd
actually investigate it.

## The setup

Imagine an email lands in a finance inbox, supposedly from a known supplier's accounts team,
asking to update the bank details for an upcoming invoice payment. It looks right at first
glance: correct logo, a signature that matches previous emails from this supplier, and it
even references a real invoice number from a recent order. The only thing pushing it into
"check this properly" territory is the tone, there's a line about needing this done today
because "the usual account is temporarily under review."

## What I actually checked

**1. The sending domain, character by character.**
I put the real supplier domain and the sender's domain side by side and read through them
slowly instead of skimming. This is the step people skip, most lookalike domains are designed
to survive a glance. In my mock version I used a single swapped character (the kind of thing
that's genuinely easy to miss if you're reading fast, an `rn` instead of `m`, or a `0` instead
of an `o`).

**2. Reply-To header.**
This is the one I think matters most and gets missed the most. Most email clients don't show
you the Reply-To address unless you go looking for it, they just show the display name. I
checked the raw headers and found the Reply-To pointed somewhere completely different from the
From address. So even if someone spotted nothing else, replying to "confirm" the new bank
details would go straight to the attacker.

**3. Domain age.**
Ran a quick WHOIS lookup (whois.com works fine for this, or `whois <domain>` from a terminal)
on the lookalike domain. It had been registered only a few days earlier. A domain that young
impersonating an established supplier is about as close to a red flag as it gets.

**4. Where the link actually goes.**
The "view invoice" link routed through a shortener before landing on a page styled to look
like a Microsoft 365 login. I didn't click through live obviously, I checked it using
urlscan.io, which lets you see where a URL redirects and what the landing page looks like
without actually visiting it yourself. The final destination was a credential harvesting page,
not a document at all.

**5. The pressure language.**
"Must be processed today," "the usual account is under review," this kind of urgency is a
deliberate tactic. It's designed to get someone to skip the normal verification step because
stopping to check feels like it risks a late payment. I've noticed this pattern come up in
basically every BEC example I've read about, it's less about the technical trick and more
about the pressure it creates.

## What I found, pulled together

- Lookalike domain, one character off, registered days before
- Reply-To pointing to a different domain entirely
- Shortened link leading to a fake Microsoft 365 login
- Urgency language designed to bypass normal checks

## What I'd actually recommend

- Block the lookalike domain and report it
- Flag it to anyone else who might have received something similar
- Any payment detail change should get confirmed by phone, using a number already on file,
  never a number or contact method from the email itself
- Push for DMARC set to `p=reject` on our own domain, so it's harder for someone to spoof us
  the same way
- Worth a short reminder to the finance team specifically about Reply-To mismatches, since
  that's the check most people don't know to make

## What this one taught me that the first project didn't

The DMARC scenario was really about trusting the protocol results over a surface read of the
email. This one was different, there was nothing technically "broken" to catch, SPF/DKIM
weren't even relevant here since the domain wasn't being spoofed at all, it was just close
enough to pass. That's what makes BEC harder to catch with automated tooling alone. A lot of
it comes down to slowing down and actually reading the headers instead of trusting the display
name, which is obvious in hindsight but easy to skip when you're moving fast through an inbox.
