---
title: "LTT attack: lessons in phishing"
date: 2023-03-26T19:40:50+05:30
draft: false
---

## primer

on 24 march, 2023, the popular tech youtuber Linus Tech Tips had his channel attacked. the attackers had taken down his existing videos, and uploaded their own. they also started a phishing livestream on his channel. after some hours, the channel was terminated, and then access was restored to him. some of his secondary youtube channels were attacked as well. 

this is one of the most high-profile phishing attacks of 2023 (hoping that it stays this way). as security enthusiasts / experts, or internet citizens in general, it is important to understand the techniques used by attackers; so we can avoid falling prey to these ourselves.

## modus operandi

this section describes the actions performed by the attackers, in chronological order. it is the `what` of this situation.

- sent a malicious sponsorship email to LTT staff.
- when the malware inside the email was (downloaded and) opened by someone on the LTT team, the software seized the session tokens inside their browsers, and sent them to the attacker.
- used these session tokens to log into LTT's channel.
- took down all old videos by setting them to unlisted.
- changed the channel name to Tesla and the channel logo accordingly.
- started a bogus stream promoting some crypto NFT grift.
- added a link to a malicious website with a cryptocurrency scam on it in the link description.

## motivation
the attackers expected that unsuspecting subscribers to LTT would check the new bogus stream, click on the link in the stream description, and pay the attackers in crptocurrency. in short, the attackers had a `financial` motivation.

## bypassed precautions
we must accept one fact here. there were many anti-malware agents and informed people who were deceived during this incident. it is typical to react to such an incident with "well they should have been more informed" or "well they should have just used 2FA" or any other such security measure.

here's a list of all the precautions that were supposed to prevent such an attack from happening:

### technical
- email spam filters.
- antivirus software, or just windows defender in the absence of any.
- authentication, multi-factor authentication to be exact.
- key required to start a stream.

### human
most technologically informed people:
- don't trust mails from spam-like sources.
- don't open shady `exe`s.
- don't trust links to scam websites.
- don't send cryptocurrency to unknown sources.
- would call out that the channel is compromised in comments.

## techniques
### email forgery

the target, a youtube creator, receives a mail from the attacker. typically, it involves a proposal from a sponsorship. these come from reputable sounding mail addresses, and contain legible sensible content. mails for this deception are created in two different ways:

- `cybersquatting` - the attacker purchases a domain with a name very similar to the real one, and sends a mail through its account. say, `@nordvpn-company.com`. alternatively, they could use a different TLD, say `@microsoft.code`.
- `spoofing` - the attacker sends a mail saying that they're from `@microsoft.com`, but sends those through an SMTP server that doesn't authenticate this mail address. gmail typically spots these forgeries, and shows a blurb showing the domain that sent the email, attaching a parameter like `via scamserver.io` to the `from` address.

### antimalware bypass

- `enlarged file size` - the infected file is enlarged by padding it with zeros or garbage data, so as to make it tougher for antimalwares to scan it.
- `anti-sandboxing` - the malware opens a pop-up with a fake error window asking for the user to click through, which stops it from being analysed in the sandbox.
- `downloading` the virus content from the internet once the process has been started. this means that the original code on the machine has no malicious code, which evades detection.

### exe cloaking

the goal here is to put an executable, while hiding this from the user. informed users typically won't trust exe files, as they are often malicious. antiviruses like windows defender will stop them from running as well, unless the user approves. the malware adapts around this in two ways

- `hidden extension` - windows hides file extensions by default. so if a file is called `notmalware.pdf.exe`, it will display the file name as just `notmalware.pdf`.
- `alternate executable extensions` - `.scr` and `.com` are executable extensions that i've seen being used for this attack. in general, don't trust an extension that you aren't already familiar with.

also, the email may include the malware in a zip file. this stops your browser from scanning the file contents while downloading it.

### session data theft

this is the meat of the attack. once you're running an executable on the victim's machine, there's a lot that you can do. here in particular, this involves stealing data from the user's browsers. in the past, this has been used to steal crypto wallet keys and passwords, but this particular attack steals `session tokens`

> **session token**
>
> you can't log into a website every time you want to access a webpage. so, websites create a `session` for you. when you log in, they give you a temporary token. you can show this token to the site prove that you had authenticated yourself in the past. your browser manages tokens these session tokens internally for you.

when a session token for a given service (like youtube) is stolen, the attacker will add that cookie to their own browser, and youtube will think that the requests are coming from an authenticated source. this allows them to bypass any single or multifactor authentication that you might have set up.

### channel takeover 

the attacker now has access to the victim's youtube channel. now they will
- `private or unlist existing videos` - this obscures evidence that this is an attacked channel, and is probably done instead of deleting to stop raising red flags on their end.
- `change channel branding` to make it look like Tesla, or whatever other company they're impersonating.
- `start a stream` - the channels run a single stream where the malicious site link is shared. sometimes they also make new videos or rebrand old ones.
- `disable comments` - on both streams and uploaded videos, all user interaction is disabled to prevent someone from calling out the scam there.

the attackers may change credentials, or run scripts to keep the attack active to prevent the victims from doing damage control. 

## why tesla, crypto, and musk?

an important question to ask here is why the attack follows _this branding_ in particular. 

using cryptocurrency is a requirement on the attacker's side. using sophisticated transaction chains the attackers can obfuscate their identity, making it tougher for authorities to crack down on them. as the transactions are done over cryptocurrency, they cannot be reversed by any regulatory authorities.

following this requirement, the branding of the attack makes more sense. keywords like `NFT`, `GPT`, `cryptocurrency`, `bitcoin`, `ethereum`, and `elon musk` are seen frequently in the titles and branding of these scams. this is targeting a vulnerable demographic, particularly those of young adults who are "enthusiasts" of decentralised finance but have little technical knowledge or foresight.

the site has dummy data as "proof" that it is in fact returning cryptocurrency to the users. these are cryptic garbage strings that give the impression of a real-time transaction table.

{{< figure src="../hashes.png" title="definitely real transaction hashes" align="center" >}}

there are many other things that the site does to gain users' trust, such as displaying a real coin price widget in the header, or having a photograph of elon musk. a sense of urgency is given to the user, by telling them that the scheme can only be availed once. these are standard tactics performed by phishing agents, and one should be wary of them.

## prescriptions

> alright, i saw all this. now what do i do about it.
- scrutinise emails from prospective sponsors. is this an official email address? look it up if it feels sketchy, or check when the domain name was registered. look out for a `via` attribute.
- enable file extensions on windows names.
- do not trust any file extension you don't already know about. most images and videos are safe (with some exceptions such as SVGs). PDFs are typically safe if you run them in your browser. i would put more trust in that than a standalone pdf reader such as adobe acrobat.
- pass the file into local antimalware or virustotal if you don't trust it.
- watch out for file sizes. no legitimate sponsorship proposal will be 800 MB or similar. most legitimate PDFs should fit under 50 MB.
- use multi-factor authentication.
- look up crypto transaction IDs to verify that they're not made up data. alternatively, inspecting the source will sometimes reveal the sloppy javascript used to create them.
- ignore the previous point and stop using cryptocurrency entirely (opinion).

## references / further reading

- linus's original video about this - https://youtu.be/yGXaAWbzl5A
- thiojoe's explanation of these cryptocurrency scams https://youtu.be/xf9ERdBkM5M
- guardio's article on streamjacking, which describes the more technical aspects of this attack - https://labs.guard.io/streamjacking-hijacking-hundreds-of-youtube-channels-per-day-propagating-elon-musk-branded-730944bbbfe6
- behavioural analysis of a similar malware from 2021, which was used to steal NFT data - https://julienvandorland.substack.com/p/the-scr-malware-hack-explained
- google threat analysis group (TAG)'s report from 2021 about youtube cookie stealing attacks - https://blog.google/threat-analysis-group/phishing-campaign-targets-youtube-creators-cookie-theft-malware/