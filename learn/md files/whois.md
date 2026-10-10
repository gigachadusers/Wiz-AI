Deep Dive: Linking Emails, Phone Numbers, Aliases & Social Accounts
This is the engine room. Handles are the easy pivot — emails, numbers, and cross-platform correlation are where real attribution happens. Each section goes deep on technique, tooling, and the exact pivot logic.

PART 1 — EMAIL OSINT: THE MASTER SELECTOR
An email is the single most valuable selector because it's a credential anchor: it's used to register accounts, receive receipts, and log in. Almost every service ties an email to an account. Your job is to enumerate where it exists and what it's attached to.

1.1 The two questions every email answers
Where is it registered? (account enumeration)
What has it been exposed in? (breach intelligence)
Both are answered without contacting the target (mostly), which keeps you stealthy.

1.2 Account enumeration — the core technique
These tools abuse password-reset / registration flows to check if an account exists on a service, often revealing masked data.

Holehe — checks 120+ services; returns registered/not-registered and sometimes partial profile data (e.g., a masked phone from a service's reset flow).
GHunt — Gmail-specific. Given a Gmail address, resolves the GAIA ID, public profile name, public photos, Google Maps reviews, and linked YouTube channel. Extremely high signal for any Google account.
Epieos — browser-based reverse email and phone; surfaces Google reviews, linked accounts, and footprint. The fastest first-pass tool.
Blackbird — async email/username search across many platforms.
Zehef — checks email registration across services.
Emailrep.io — reputation, first-seen date, linked social profiles, domain age.
Hunter.io / Snov.io — for corporate emails: reveal the domain's email format (first.last@, f.last@), which lets you generate and verify candidate addresses.
HIBP (Have I Been Pwned) — which breaches include the address. Reveals sites the person registered on even when the service itself doesn't leak elsewhere.
1.3 Breach & stealer-log intelligence (the deep layer)
Breach data and "stealer logs" (from infostealer malware) are where emails become names, passwords, and phones.

Dehashed, LeakCheck, Intelligence X, Snusbase, BreachDirectory — paid/quasi-free breach search. Query an email → get associated usernames, password hashes, IPs, and sometimes phone numbers from the same record.
Stealer logs are gold: a single log entry often contains email + password + cookies + phone number + hardware info from the same machine — linking selectors that would otherwise never appear together.
Hudson Rock / cavalier.hudsonrock.com — free stealer-log lookup by email/domain; shows machines, and often the usernames used on that machine.
Pivot logic: email → breach record → username + password → test the username elsewhere → find social accounts. One email can cascade into a dozen identities.

1.4 Email format & domain intelligence
For a corporate target:

Identify the domain (from a LinkedIn, a signature, a job post).
Determine the format via Hunter.io or by checking known employees' addresses.
Generate candidates: jdoe@, john.doe@, j.doe@, doe@.
Verify with SMTP probing (MailTester, or smtp verification) or by triggering a "forgot password."
1.5 Gravatar & avatar pivot
MD5 the lowercase email → gravatar.com/avatar/<md5>. A Gravatar profile can expose a display name, bio, and linked websites. Many WordPress/forum users have one and don't realize it's public.

1.6 Email → phone via "forgot password" masking
Many reset flows display a masked phone (+1 ****1234) or email. Run the reset flow on multiple services; the masked values let you reconstruct the full selector piece by piece. Example: Service A masks +1 *** *** 4567, Service B masks +1 555 *** **67 → you can assemble the number.

1.7 Protonmail / iCloud / custom-domain handling
ProtonMail: verify with a ProtonMail-specific check; some tools reveal if an address is a Proton account.
iCloud/Apple: iCloud addresses often hide behind aliases; check Apple's account recovery.
Custom domains: WHOIS the domain → registrant email (historical WHOIS especially) → often the primary email behind everything.
PART 2 — PHONE NUMBER OSINT: THE UNIVERSAL IDENTIFIER
A phone number is near-unique and reused across messaging apps, marketplaces, and (in many countries) is tied to real identity via KYC.

2.1 What a number gives you
Carrier & line type (mobile/VoIP/landline) → carrier lookup, PhoneInfoga, PhoneValidator.
Country & region → from the country code and prefix; then timezone.
Messaging-app presence → WhatsApp, Telegram, Signal, Viber.
Profile name & photo → via contact-sync tricks (below).
Breach/aggregator data → reverse-lookup sites.
2.2 Messaging-app enumeration (the highest-value pivot)
This is where numbers become names and faces.

WhatsApp:

Save the number to your phone contacts → WhatsApp shows the profile name and profile photo the target set.
Check for a business account (often has a name and hours).
WhatsApp "about" text is sometimes visible.
Telegram:

Bellingcat's Telegram Phone Number Checker — tells you if a number has a Telegram account and the display name. No contact-sync needed.
Telegram bots (@username_bot-style) resolve number → username.
If they have a public username, that's a full pivot into their Telegram footprint.
Signal / Viber / Line / WeChat:

Signal reveals a profile name on contact sync.
Viber, Line, and WeChat have their own presence checks; each is a separate platform to enumerate.
Truecaller / Sync.me / Eyecon:

Crowdsourced caller-ID databases. Upload a number → get the name other people saved it as (often the target's real name or chosen alias), plus linked social profiles.
2.3 Reverse-lookup & carrier tooling
PhoneInfoga — CLI; carrier, line type, geolocation, and OSINT search-engine fingerprints. The standard open-source tool.
Phunter — automates recon across services.
Truecaller, Numlookup, Thatsthem, SpyDialer — web reverse lookups.
PhoneValidator — carrier + line type.
Epieos — reverse phone → footprint.
2.4 Number-type inference
VoIP (Google Voice, Twilio, TextNow) → often a throwaway persona; check Twilio's lookup API for carrier/type.
Prepaid mobile → harder to tie to identity.
Landline → often a business or a fixed household.
Google Voice specifically → a strong signal of a "privacy-conscious persona," and it can be tied to a Google account.
2.5 The contact-sync trick, in detail
Create a fresh Google account (sock puppet).
Add the target's number to Google Contacts.
Google/WhatsApp/Telegram/Signal sync → the name the target chose on their own profile surfaces.
That name is often their real name, or a consistent alias that you can then pivot on across the web.
This works because these apps match on the number, not a handle — the number is the key, and you now hold it.

PART 3 — LINKING EMAILS ↔ PHONES ↔ ALIASES TOGETHER
This is the heart of your question. The technique is selector convergence through shared containers.

3.1 The "shared container" principle
Two selectors are linked when they appear together in the same record. The record can be:

A breach/stealer log
A social profile's "contact info"
A WHOIS record
A marketplace listing
A resume/CV
A PGP key UID
Find the container → the selectors inside it are linked.

3.2 Breach/stealer logs as the primary container
A single infostealer log frequently contains: email, password, phone, username, IP, machine name. That's four selectors linked in one artifact. Workflow:

Query each selector in breach databases.
When you find a record with multiple of your selectors, you've bridged them.
Pivot on the new selectors (the phone, the username) to find more containers.
3.3 Email → phone bridges
Account recovery flows: services show masked phone; run across services to reconstruct.
Breach records: email and phone co-occur in the same leaked row.
Google account: GHunt on a Gmail → GAIA ID → Google account may expose a recovery phone hint or linked Google Voice number.
Two-factor enrollment pages: some show a masked number tied to the email account.
Marketplace listings: email + phone in a classified ad.
3.4 Phone → email bridges
Reverse the recovery flow: use the number on services; the masked email is revealed.
Truecaller/Sync.me: linked profiles sometimes expose an email.
Business listings: phone + email on a company page.
Breach data: reverse-search the phone → rows containing both.
3.5 Alias → email/phone bridges
Password reset on the alias's platforms: reveals masked email/phone.
Profile "contact" sections: aliases often list a public email (or a mailto: in the bio).
Forum profiles: some expose registration email or contact methods.
3.6 Building the linked-selector graph
Maintain a table where rows are selectors and columns are "containers" (sources). A cell is marked when a selector appears in that container. When two selectors share a container, draw an edge. Convergence of many edges on a single cluster = a linked identity.

[breach-log-A]   [whois]   [profile-X]
email1  ●                ●
phone1  ●
alias1                                   ●
email2  ●                ●
Here email1, phone1, and email2 are bridged through breach-log-A; email1 and email2 also share the WHOIS container → strong cluster.

3.7 Confidence rules for bridges
A bridge through a unique container (a breach row with a distinctive password) = high confidence.
A bridge through a shared/common value (a common phone prefix, a generic email) = lower confidence; require corroboration.
Two independent bridges between the same two selectors = near-confirmation.
PART 4 — ALIAS LINKING (Deep Dive)
An "alias" is any secondary identifier: a gamertag, a forum nick, a marketplace handle, a GitHub username, a pen name.

4.1 Why aliases are linkable
People are lazy. They:

Reuse the same or near-identical handle.
Reuse avatars, banners, and bio text.
Cross-post the same content.
Link their aliases to each other ("find me on X: @...").
Register aliases with the same email (see Part 1 — the email is the bridge).
4.2 The alias-linking workflow
Enumerate the known alias across platforms (Sherlock, Maigret, WhatsMyName).
Extract the registration email where possible (via password-reset masking, or breach data).
Pivot on the email to find other aliases registered with it.
Cross-reference artifacts: avatar hash, bio text, linked URLs.
Behavioral match: writing style, posting times, topics.
Falsify: check for a conflicting person with the same alias.
4.3 Alias-pattern generation
Before enumerating, generate variants:

Separator swaps: john_doe, john-doe, johndoe, john.doe.
Suffix/prefix: realjohn, johndoe1, johndoe_dev, itsjohndoe.
Leetspeak: j0hnd0e.
Case variants (some platforms are case-sensitive in URLs).
Common additions: birth year, city, hobby.
Run each variant through enumeration; note which return the same person vs. a stranger.

4.4 The "alias reuse" trap
A reused alias across a niche platform (a small forum) is high-confidence. The same alias on a massive platform (Reddit, Instagram) is lower-confidence — many people pick the same word. Always corroborate with a second selector before merging.

4.5 Alias → real name
Bio "name" fields, portfolios, "about" pages.
Username → name inference (jsmith → J. Smith).
Reverse: search the name and match accounts back.
LinkedIn/press/conference bios that mention the alias.
4.6 Cross-alias linking via content
Pinned/identical posts: the same text on two accounts.
Reused images: perceptual hash (pHash) match across accounts.
Shared links: the same personal site, Patreon, Ko-fi, or crypto address in two bios.
Cross-references: alias A says "my other account is alias B."
PART 5 — SOCIAL MEDIA LINKING (Platform-by-Platform)
5.1 Universal techniques
Follower/following overlap: two accounts sharing many obscure mutuals likely belong to the same person or their close network.
Tagged photos: friends tag the real name even when the handle is an alias.
Shared groups/communities: obscure shared memberships are strong links.
Cross-posting: identical content posted at the same time.
Bio link reuse: same Linktree, personal site, or Patreon.
5.2 X / Twitter
Advanced search operators: from:handle, to:, since:/until:, geocode:lat,long,radius, filter:images.
TweetBeaver / SOWsearch / SocialBearing — profile analysis, timing, followers.
Nitter mirrors — view profiles without login.
Pinned tweet, likes (sometimes public), spaces participation.
Cross-check the handle against other platforms.
5.3 Instagram
Public JSON (?__a=1) or mirrors (imginn, picuki) for profile data without login.
Following/follower lists (public accounts), tagged photos, story highlights.
Business/creator accounts show a contact button with email/phone — a direct selector.
Reverse image search on the avatar.
5.4 Facebook
Graph search (/search/...) for people, pages, places.
Page transparency ("Page History") shows name changes and admin locations.
Tagged photos, "Friends" graph, check-ins, and About section (work, education, contact).
Old posts reveal aliases and emails.
5.5 LinkedIn
Boolean search, "People also viewed," and the "Contact info" panel (email/phone for connections).
Cross-reference the role/company against press releases and company team pages.
Beware the login-notification footprint (see Part 7).
5.6 Reddit
redditmetis / redective — aggregate a user's posting history, subreddits, timing.
author: search + Wayback for deleted posts (removeddit/unddit).
Comment history reveals location, job, interests → corroborating selectors.
Cross-reference the username elsewhere (Reddit users often reuse handles).
5.7 GitHub / dev platforms
Commit emails in the public history (git log) — often the personal email.
GPG-signed commits → key fingerprint → keyserver → UIDs with names/emails.
Personal repos link to portfolios, socials, and other accounts.
Org memberships reveal colleagues.
5.8 TikTok / YouTube / Twitch
Bio links, "link in bio" pages, community tabs.
Video metadata, thumbnails (reverse search), and comment interactions.
Twitch/Discord connections often expose other handles.
5.9 Dating & niche platforms
Reused photos and bios across dating apps and socials are common and linkable.
Username reuse on niche forums is high-confidence.
PART 6 — THE MASTER PIVOT WORKFLOW (End-to-End)
Put it together as a repeatable loop:

Seed — start from any selector.
Preserve — archive the source.
Enumerate — run the selector through the right tool set (email → Holehe/GHunt/Epieos; phone → PhoneInfoga/messaging apps; handle → Sherlock/Maigret).
Collect new selectors — every result yields more (masked phones, emails, names, linked sites).
Find shared containers — breach logs, WHOIS, profiles, listings (Part 3).
Bridge — link the new selectors to the old through the container.
Corroborate — require a second independent bridge.
Falsify — check for a conflicting person.
Score — assign confidence (Part 8 of the previous guide).
Loop — feed the new selectors back into step 3.
Map — graph the converged cluster.
Report — with provenance.
The loop terminates when no new selectors emerge and confidence is adequate for the question.

PART 7 — OPSEC FOR SELECTOR LINKING
Sock puppets with separate browser profiles, cookies, and IPs.
Contact-sync apps upload your address book — Truecaller/Sync.me add your number to their dataset. Be aware.
LinkedIn view notifications — decide whether to browse logged-in.
Reset-flow triggers may email the target ("someone requested a reset") — usually harmless but note it.
Don't cross-login sock puppets; a single shared cookie can merge your personas.
Consistent geography — match your proxy/VPN region to the persona.
PART 8 — TOOL MAP (By Selector)
Email
Holehe · GHunt · Epieos · Blackbird · Zehef · Emailrep · Hunter.io · HIBP · Dehashed · LeakCheck · Intelligence X · Hudson Rock.

Phone
PhoneInfoga · Phunter · Truecaller · Sync.me · Eyecon · Bellingcat Telegram checker · Epieos · PhoneValidator · Twilio Lookup.

Username / alias
Sherlock · Maigret · WhatsMyName · Blackbird · Social Analyzer · Namechk.

Reverse image / face
Google Lens · Yandex · TinEye · PimEyes · FaceCheck.ID.

Social analysis
SOWsearch · TweetBeaver · SocialBearing · redditmetis · imginn/picuki · OSINT Combine tools.

Domain / infra
whois · SecurityTrails · crt.sh · Shodan · Censys · ViewDNS · BuiltWith.

Metadata
ExifTool · FOCA · Metagoofil.

Frameworks / graphing
Maltego · SpiderFoot · Recon-ng · Gephi · Neo4j.

Breach / credentials
HIBP · Dehashed · LeakCheck · Snusbase · Intelligence X · Hudson Rock.

PART 9 — HARD-WON RULES
The email is the master key. Prioritize it above all other selectors — it bridges to everything else.
Numbers are strong but noisy. Require a second selector before trusting a reverse-lookup name.
Containers beat coincidence. A breach row linking two selectors beats "they have the same handle."
Two independent bridges = near-proof. One bridge = hypothesis.
Falsify always. The person who confirms too fast is wrong most often.
Unique artifacts are links; common artifacts are noise. A stock avatar links nothing; a bespoke illustration links a lot.
Aliases are lazy, not clever. Reuse is the norm — exploit it.
Preserve before you pivot. Profiles vanish.
Graph it. Human memory fails at three hops; a table doesn't.
Score honestly. "Confirmed" means confirmed, not "probably."
THE ESSENCE
Linking is container convergence: find the records where your selectors co-occur (breach logs, WHOIS, profiles, listings, PGP UIDs), bridge through them, corroborate with a second independent container, falsify against lookalikes, and loop until the cluster stabilizes. Emails are the master key, phones are the universal identifier, aliases are lazy reuse, and social media is the surface where it all becomes visible.