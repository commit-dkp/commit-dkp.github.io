---
title: "xkcdcrypt"
description: "A proof-of-concept file encryption tool for actual humans"
---
*This post was the foundation for a talk I gave at the [DefCon Crypto & Privacy Village](https://cryptovillage.org/),
so if you are a more visual learner, then please enjoy 
[the stream](https://www.youtube.com/watch?v=4WWvvn__wEw&t=11098s).*

I had a problem one day. A problem you might have had, too, and assuredly we all have eventually. I needed to send 
someone some confidential documents, and it had to be via email. Email, however, is one of the least secure ways of 
sending anything. It is stored and forwarded by an unknowable number of intermediaries between the sender and 
recipient, like a letter read and copied by every postal employee who handles it. The fact that there might be some 
transport layer security in use doesn't change the fact that the security is only 
[point-to-point](https://www.ndss-symposium.org/ndss2017/ndss-2017-programme/security-impact-https-interception/) 
rather than [end-to-end](https://signal.org/docs/specifications/x3dh/), the letter's envelope opened and replaced at 
every exchange.

Anyways, this someone wasn't about to click any links to a file hosting site, nor were they about to install any 
third-party encryption software. They worked at the kind of place that had policies against doing those sorts of 
things, y'know. Just as well, for the former only makes clicking the link a [race](https://send.firefox.com/) between 
them and all intermediaries, and the latter is frankly a wasteland of the [unusable](https://arxiv.org/abs/1510.08555) 
and the [untrustworthy](https://www.usenix.org/conference/usenixsecurity19/presentation/muller). What, then, can one do 
using only what's already available?

Start by compromising, because for the common zip file format, macOS and Windows 
[don't](https://devblogs.microsoft.com/oldnewthing/20180515-00/?p=98755) ship strong encryption. Instead, they still 
ship the PKZIP stream cipher, a rolled-their-own algorithm that is both 
[broken](http://www.cs.technion.ac.il/users/wwwb/cgi-bin/tr-get.cgi/1994/CS/CS0842.pdf) in 2^38 operations with 13 
consecutive bytes of known plaintext and 
[cracked](http://www.insticc.org/Primoris/Resources/PaperPdf.ashx?idPaper=73605) at a rate of 1.5 * 10^9 password 
candidates per second. That all means Apple and Microsoft's encryption for zip files is worse than useless, instilling 
a false sense of security that stems the demand for strong encryption. Pragmatically, though, even weak cryptography 
will challenge opportunistic intermediaries, so one soldiers on.

Further compromise is required because how does one share the zip file's password with the recipient? Can't send it 
over email, of course. If there were another, secure channel to send the password over, then one might skip the whole 
encryption circus and simply send the documents that way. Alternatively, one could estimate the improbability of both 
the recipient's email *and* phone being compromised, but then have fun reading off a randomly generated password like 
`q|}p"%,+5H` to them. A text message might be easier, but don't count on any thanks for making them type that in.

The thing is, one can string six random, real words together for a passphrase that's even stronger than one that looks 
like a cat danced on the keyboard. With a US standard keyboard, it's possible to express over 53 quintillion different 
10-character passwords and, at a rate of over 47 quadrillion attempts per year, it would take well over 5 centuries to 
test even half of them. With a wordlist of 2048 words, it's possible to express over 73 quintillion different 6-word 
passphrases, and at the same rate it would take well over 7 centuries. A rare win-win, where a secret like 
`burger-artist-bike-degree-decline-mouse` is both harder to guess and easier to share. If you wouldn't mind a 12-word 
passphrase, you could turn a cryptographically random 128-bit value suitable for encryption into something more 
suitable for humans with the [BIP39](https://github.com/bitcoin/bips/blob/master/bip-0039.mediawiki) standard.

Crucial to all that math is the assumption of randomness, and yet the responsibility to choose a password is always 
delegated to people. People are so predictable in their password choices that anyone can download the 100 thousand most 
[common](https://www.ncsc.gov.uk/static-assets/documents/PwnedPasswordTop100k.txt) passwords. We know they're common 
because they keep showing up in data [exfiltrated](https://haveibeenpwned.com/) from compromised sites. As these lists 
demonstrate, people are not naturally disposed to meet the requirements of the security systems they're forced to 
depend on. And when they do manage to meet the artificial requirements forced upon them by all the 
[byzantine](https://www.wsj.com/articles/the-man-who-wrote-those-password-rules-has-a-new-tip-n3v-r-m1-d-1502124118) 
password policies, they do so with a vengeance, reusing the same password everywhere. We know they're reused because 
the sites themselves have been compromised in this way, when 
[employees](https://ico.org.uk/about-the-ico/news-and-events/news-and-blogs/2018/11/ico-fines-uber-385-000-over-data-protection-failings/) 
reused passwords on other sites that were compromised. The more secure, more usable system is one that does not expect 
people to behave like computers.

To that end, I've written [xkcdcrypt](https://github.com/commit-dkp/xkcdcrypt) as a proof-of-concept file encryption 
tool, named in honor of the xkcd webcomic on [password strength](https://xkcd.com/936/). xkcdcrypt addresses the broken 
cryptography problem by adopting the GCM-SIV mode of the AES cipher. xkcdcrypt addresses the password cracking problem 
by adopting the Argon2d key derivation function. But perhaps most importantly, xkcdcrypt addresses the credential 
stuffing and password spraying problem by *not letting the user choose the password*.

The advantages of [AES-GCM-SIV](https://cyber.biu.ac.il/aes-gcm-siv/) over the PKZIP stream cipher are too many to 
describe here, but it is worth highlighting the nonce-reuse-misuse-resistance advantage of the GCM-SIV mode over other 
modes of the AES cipher. As the name implies, nonces are cryptographic values that must be used only once per 
key-message combination, else important security guarantees are not met. In other modes of AES, nonce reuse results in 
[catastrophic](https://eprint.iacr.org/2016/475) failure, enabling message forgery and plaintext recovery. In 
AES-GCM-SIV, the failure is limited to ciphertext 
[distinguishability](https://blog.cryptographyengineering.com/why-ind-cpa-implies-randomized-encryption/), only 
enabling the observation that the same plaintext was encrypted more than once with the same key and nonce.

The advantages of [Argon2](https://www.cryptolux.org/index.php/Argon2) over the 
[CRC](http://reveng.sourceforge.net/)-based key derivation function used by the PKZIP stream cipher are also too many 
to describe here, but it is worth highlighting the particular resistance Argon2 has to GPU-based password cracking. 
Because Argon2 uses data-dependent memory access, one can't simply throw more RAM at it to make it go faster. Where as 
a GPU-based implementation of the Zip algorithms can try 1.5 * 10^9 passwords per second, a GPU-based implementation of 
Argon2 can only try about 308 passwords per second. That means, even if one shrinks the passphrase down to 4 words, it 
would still take over 9 centuries to test even half of the 17 trillion or so possible passphrases.

*Not letting the user choose the password*, though, is the most significant advantage because a common or reused 
password has no defensive significance whatsoever. Given the ephemeral nature of the security challenge, to email some 
files to another party and separately text them the passphrase, this approach works well enough. This approach does not 
work so well for persistent security challenges, like for example repeatedly authenticating to a web site. Sites that 
would adopt this same approach will quickly find many of their users are unprepared to save the issued passphrase in a 
password manager. It's still an open question whether it might be easier for sites to encourage adoption of 
[password managers](https://keepassxc.org/) or adoption of potentially passwordless authentication like 
[WebAuthN](https://webauthn.guide/), but there's no question that relying on passwords chosen by people is asking too 
much of people and too little of computers.

If you need to explore this problem space further, like Senator Ron Wyden 
[asked](https://www.wyden.senate.gov/imo/media/doc/061919%20Wyden%20Sensitive%20Data%20Transmission%20Best%20Practices%20Letter%20to%20NIST.pdf) 
NIST to do, then look in the direction of [Magic Wormhole](https://github.com/warner/magic-wormhole),
[sear](https://github.com/iqlusioninc/sear), and the 
[AGE](https://docs.google.com/document/d/11yHom20CrsuX8KQJXBBw04s80Unjv8zCg_A7sPAX_9Y/edit) specification. Magic 
Wormhole is a neat approach to transferring files from one computer to another using human-pronounceable wormhole 
codes. It's also way easier than bootstrapping [scp](https://help.ubuntu.com/community/SSH), though it does require 
both computers to have Magic Wormhole installed and a server to deliver messages from one to the other. While presently 
self-described vaporware, and open to the use of chosen passwords, sear will be a tool and library for creating 
always-encrypted file archives. The AGE specification, its support for user-chosen passwords notwithstanding, is 
ultimately intended to be a replacement for [GPG](https://blog.filippo.io/giving-up-on-long-term-pgp/)-based file 
encryption but with public keys that are easier to share. At this time, it is only a specification, but ambitious 
readers are encouraged to start a reference implementation.

The problems we face in encrypting files to each other are largely due to a lack of vendor willpower. The problems that 
remain in managing the password, however, are due to a lack of human empathy. In their own way, though, xkcdcrypt and 
these other projects show there's still much we can do to improve security by centering the design on people.

*Thanks to [Tony Arcieri](https://tonyarcieri.com/) and [JP Aumasson](https://aumasson.jp/).*
