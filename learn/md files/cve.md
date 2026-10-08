Absolutely. One correction that’s useful for teaching an AI: a CVE is not the bug itself. A CVE is a standardized identifier for a publicly known vulnerability. The actual flaw might be command injection, memory corruption, authentication bypass, XSS, deserialization, etc.

Below is a 5-per-year teaching set for 2024, 2025, and 2026. I’ve kept the exploitation descriptions conceptual rather than giving weaponized payloads.

2024
1. CVE-2024-3094 — XZ Utils backdoor

Type: Supply-chain compromise / malicious code
Severity: Critical

How it works: Malicious code was inserted into XZ Utils 5.6.0 and 5.6.1. The build process extracted hidden code from disguised test material and modified liblzma. On affected systems, this could interfere with software using the library, including SSH-related functionality.
How it was found: Andres Freund noticed unusual SSH performance while investigating a performance problem and traced it to the compromised XZ package.
How to patch: Remove/replace XZ Utils 5.6.0/5.6.1 with a trusted, unaffected version supplied by the OS vendor. Because this was a supply-chain compromise, simply rebuilding an application isn't enough—you need to verify the underlying package.
AI lesson: A vulnerability doesn't always originate from a programming mistake. Compromised dependencies can introduce malicious behavior into otherwise legitimate software.
2. CVE-2024-3400 — Palo Alto PAN-OS

Type: Command injection / arbitrary file creation
Severity: Critical, CVSS 10.0

How it works: A flaw in the GlobalProtect functionality could let an unauthenticated attacker cause arbitrary file creation and ultimately execute commands with root privileges on affected PAN-OS firewalls.
How it was found: The issue was publicly disclosed through Palo Alto Networks' security response process and was subsequently added to CISA's Known Exploited Vulnerabilities catalog.
How to patch: Upgrade affected PAN-OS versions to the vendor-provided fixed releases. Palo Alto also provided temporary mitigations/Threat Prevention protections while patches were being deployed.
AI lesson: Input that reaches an operating-system command interpreter must be strictly controlled. Treating attacker-controlled data as a command is fundamentally different from treating it as data.
3. CVE-2024-4577 — PHP-CGI on Windows

Type: OS command injection
Severity: Critical, CVSS 9.8

How it works: On certain Windows configurations, Windows' character conversion ("Best-Fit" behavior) could transform characters in a request into something PHP-CGI interpreted as command-line options. That could allow source-code disclosure or arbitrary PHP execution.
How it was found: Security researchers investigated the issue and subsequently researchers such as Bitsight analyzed ways to reliably detect affected installations.
How to patch: Upgrade PHP to 8.1.29, 8.2.20, or 8.3.8, depending on the branch.
AI lesson: Security bugs can happen at the boundary between components—in this case HTTP input → Windows encoding → command-line parsing → PHP.
4. CVE-2024-6387 — OpenSSH "regreSSHion"

Type: Race condition / memory corruption / remote code execution
Severity: High, CVSS 8.1

How it works: A race condition involving OpenSSH's signal handling could potentially corrupt memory and lead to remote code execution against vulnerable sshd installations. It was also a regression of an older vulnerability, meaning a previously fixed security property had effectively returned.
How it was found: The Qualys Threat Research Unit discovered and analyzed the vulnerability, identifying it as a regression of CVE-2006-5051.
How to patch: Upgrade OpenSSH through your operating-system vendor's security updates. The vulnerability was addressed in OpenSSH 9.8 and corresponding downstream patches.
AI lesson: Regression bugs matter. A security property that was fixed years ago can accidentally be broken again by later code changes.
5. CVE-2024-21410 — Microsoft Exchange Server

Type: Authentication / privilege escalation
Severity: Critical, CVSS 9.8

How it works: A vulnerability in Exchange Server could allow privilege escalation and was significant enough to be placed in CISA's Known Exploited Vulnerabilities catalog.
How it was found: Microsoft identified and disclosed the vulnerability through its security response process.
How to patch: Apply Microsoft's Exchange security updates corresponding to the affected cumulative updates. CISA specifically recommended applying the vendor mitigation or discontinuing use where mitigation wasn't available.
AI lesson: Authentication failures and privilege escalation are different concepts from RCE, but can still result in complete compromise depending on what privileges are gained.
2025
1. CVE-2025-29927 — Next.js middleware authorization bypass

Type: Authentication/authorization bypass
Severity: Critical

How it works: Next.js applications that relied on middleware for authorization could have their authorization checks bypassed under certain conditions.
How it was found: The vulnerability was disclosed through the Next.js/GitHub security-advisory process, with fixes committed to the project.
How to patch: Upgrade to:
Next.js 12.3.5
13.5.9
14.2.25
15.2.3
Workaround: If patching wasn't immediately possible, filtering externally supplied x-middleware-subrequest headers was recommended.
AI lesson: Authentication ≠ authorization. An application can correctly identify a user but still incorrectly decide that user is allowed to access something.
2. CVE-2025-32433 — Erlang/OTP SSH RCE

Type: Missing authentication / remote code execution
Severity: Critical, CVSS 10.0

How it works: Erlang/OTP's SSH implementation improperly processed certain SSH protocol messages before authentication. An attacker could therefore reach functionality that should only have been accessible after successful authentication.
How it was found: Researchers Fabian Bäumer, Marcus Brinkmann, Marcel Maehren and Jörg Schwenk from Ruhr University Bochum reported finding the flaw.
How to patch: Upgrade to:
OTP 27.3.3
OTP 26.2.5.11
OTP 25.3.2.20
Temporary mitigation: Disable the Erlang SSH server or firewall access to it until patching is possible.
AI lesson: Authentication checks must happen before privileged protocol functionality is processed.
3. CVE-2025-49113 — Roundcube Webmail

Type: PHP object deserialization / RCE
Severity: High/Critical depending on scoring methodology

How it works: An authenticated user could abuse insufficient validation of the _from parameter. The vulnerable code path could result in PHP object deserialization and ultimately remote code execution.
How it was found: The public record references research by Fearsoff, along with the corresponding Roundcube fixes.
How to patch: Upgrade Roundcube to:
1.5.10, or
1.6.11.
AI lesson: Deserialization is dangerous when attacker-controlled data can become executable object state.
4. CVE-2025-5777 — Citrix NetScaler "CitrixBleed 2"

Type: Memory over-read / information disclosure
Severity: Critical

How it works: Insufficient input validation could cause NetScaler to read beyond the intended bounds of memory when configured for particular Gateway/AAA functionality.
How it was found: Citrix disclosed the issue through its security advisory process; the CVE record itself does not establish the original researcher's identity.
How to patch: Upgrade affected NetScaler releases to the fixed versions, including:
14.1 → 43.56 or later
13.1 → 58.32 or later
AI lesson: Memory safety bugs don't have to immediately provide RCE. A read-out-of-bounds bug can first expose sensitive memory and potentially become part of a larger attack chain.
5. CVE-2025-55182 — React Server Components / "React2Shell"

Type: Unsafe deserialization / remote code execution
Severity: Critical, CVSS 10.0

How it works: React Server Components accepted HTTP requests containing serialized data. The vulnerable implementation did not safely handle that untrusted serialized data, allowing an unauthenticated attacker to potentially reach remote code execution.
How it was found: It was publicly disclosed by the React/Meta security team on December 3, 2025; the CVE record links to the React security advisory.
How to patch: Upgrade affected React Server Components packages to vendor-fixed releases. Applications using affected Next.js versions also needed the corresponding Next.js security update.
AI lesson: This is another good example of CWE-502: deserialization of untrusted data—a useful category for an AI to recognize across completely different software projects.
2026

2026 is still ongoing, so these are vulnerabilities that have been publicly recorded so far in 2026, rather than pretending the year's CVE list is complete.

1. CVE-2026-10520 — Ivanti Sentry

Type: OS command injection / remote code execution
Severity: Critical, CVSS 10.0

How it works: An unauthenticated remote attacker can exploit OS command injection in affected Ivanti Sentry versions to execute commands with root-level privileges.
How it was found: Ivanti disclosed the issue; public research/advisory material also appeared around the vulnerability, including analysis by watchTowr.
How to patch: Upgrade to:
R10.5.2
R10.6.2
R10.7.1, depending on your branch.
AI lesson: Command injection occurs when data controlled by an attacker crosses a boundary into an OS command interpreter without sufficient neutralization.
2. CVE-2026-26105 — Microsoft SharePoint

Type: Cross-site scripting (XSS)
Severity: High/Critical depending on scoring

How it works: Improper input neutralization in SharePoint allows malicious web input to be interpreted as active content rather than harmless data. The CVE describes spoofing and impacts to confidentiality/integrity.
How it was found: Microsoft reported the vulnerability through its security response process; the CVE was published March 10, 2026.
How to patch: Apply Microsoft's security updates. Affected builds include:
SharePoint 2016 below 16.0.5543.1000
SharePoint 2019 below 16.0.10417.20102
Subscription Edition below 16.0.19725.20076.
AI lesson: XSS = untrusted input becoming executable browser content. Proper output encoding/context-aware sanitization prevents this class of bug.
3. CVE-2026-26046 — Moodle

Type: Command injection
Severity: High, CVSS 7.2

How it works: Insufficient sanitization of an administrative TeX-filter setting could allow command injection when the TeX filter and ImageMagick are configured in the affected way. Exploitation requires administrative privileges.
How it was found: The vulnerability was reported through the Moodle/Fedora security ecosystem; the CVE was published February 21, 2026.
How to patch: Upgrade to:
Moodle 4.5.9
5.0.5
5.1.2, as appropriate.
AI lesson: Privilege requirements matter. A command-injection vulnerability requiring an administrator is still serious, but it has a very different attack path from an unauthenticated RCE.
4. CVE-2026-31694 — Linux kernel FUSE

Type: Kernel memory corruption / out-of-bounds write
Severity: High, CVSS 7.8

How it works: FUSE could receive a directory entry whose serialized size was larger than a memory page. The kernel checked whether the entry fit in the remaining page space but failed to check whether the entry itself exceeded PAGE_SIZE. That could cause a memcpy() to overwrite the following kernel page.
How it was found: The Linux kernel development/security process identified the flaw and produced a kernel fix that explicitly rejects oversized directory entries.
How to patch: Install the kernel updates supplied by your Linux distribution. Ubuntu, for example, incorporated the fix through its 2026 security updates.

AI lesson: This is a great example of memory-bound checking:
attacker-controlled length → calculated size → bounds check → memory copy.

Missing one validation step can turn ordinary input parsing into kernel memory corruption.

5. CVE-2026-25645 — Python Requests

Type: Insecure temporary-file handling
Severity: Medium, CVSS 4.4

How it works: requests.utils.extract_zipped_paths() could use a predictable filename in the system temporary directory. A local attacker able to manipulate that directory could pre-create a malicious file which the vulnerable code would reuse.
How it was found: The issue was reported through the Python Requests/GitHub security-advisory process and assigned CVE-2026-25645.
How to patch: Upgrade Requests to 2.33.0 or later. The fixed implementation uses a non-deterministic extraction location.
AI lesson: Temporary files are security-sensitive. Predictable names + attacker-controlled directory access = potential file substitution attack.
The pattern your AI should learn

You can teach your AI to represent every CVE using something like this:

Field	Meaning
CVE ID	Unique identifier, e.g. CVE-2026-10520
Year	Year assigned/published
Product	Software affected
Vulnerability class	RCE, XSS, SQLi, auth bypass, memory corruption, etc.
Attack requirements	Network/local, authentication required, user interaction, privileges
Impact	Confidentiality, integrity, availability
Root cause	The actual programming/design mistake
Discovery	Researcher, vendor, audit, incident, fuzzing, code review, etc.
Disclosure	When it became publicly known
Patch	Fixed software version
Mitigation	Temporary protection if a patch isn't immediately possible
Detection	How defenders can determine whether they're exposed
CWE	General weakness category
One particularly important distinction

Your AI should not learn:

"CVE-2025-32433 = RCE."

It should learn:

CVE-2025-32433 is an identifier for a specific Erlang/OTP SSH vulnerability. The underlying weakness involved processing SSH protocol messages before authentication, resulting in unauthenticated RCE. The fix was implemented in specific OTP releases.

That distinction is important because CVEs identify individual vulnerabilities, while CWE describes the underlying weakness category, and CVSS describes severity/risk characteristics.

For example:

CVE → specific vulnerability
CWE → type of programming/security weakness
CVSS → severity/risk score
CPE → affected product/version identification

That gives your AI a much more accurate mental model of vulnerability management.