# Fingerprinting :: moving forward and away from b0rked SHA1

Okay, it's a known fact that the SHA1-based fingerprinting used in qiqqa to identify PDF documents is flawed: that's a bug that exists since the inception of the software. The SHA1 binary hash was encoded as HEX, but any byte value with a zero(0) for the most significant nibble would have that nibble silently discarded.

This results in a couple of things, none of them major, but enough for me to consider moving:

- the SHA1 hash is thus corrupted as-useed in Qiqqa: the nibble sequence `0A BC 0D` in a hash is indistinguishable from `AB 0C 0D` and `0A 0B CD` as all encode as `ABCD`. Thus the number of 'bits' in the hash are reduced quite a bit - not enough to cause immediate disaster, but there's more risk at collisions as the hash is not used the way it should.
- if I want to store both SHA1-colliding PDFs as separate documents, I can't. That's not important if I only store research papers, but when you are stretching the original goals of qiqqa to suit your needs, you're in trouble.
 

Given the b0rked SHA1 encoding in Qiqqa, you can expect more than 1 collision out there as thee number of hash bits is close to *halved* due to the encode b0rk, going from 160bits to ~80 bits and that's getting close to 'hmmm, not feeling all that safe anymore' chances: https://en.wikipedia.org/wiki/Birthday_problem#Probability_table as I'm looking for a fingerprint that can virtually *guarantee* uniqueness of fingerprint per document. In that table, I'm therefor only interested in the p <= 10<sup>-18</sup> column as I'm risk averse like that.

## So first question then is: can we grab ourselves another hash, which does better?

Sure we can. Plenty of them!


## But how about fingerprinting (i.e. *hashing*) *performance*?

Bigger hashes tend to get slower. but then I ran into this one: https://crypto.stackexchange.com/questions/26336/sha-512-faster-than-sha-256 (SHA512 faster than SHA256) and that got me looking for more.

There's also the article [Too Much Crypto by Aumassen](https://eprint.iacr.org/2019/1492.pdf), which addresses the uniqueness ('collision') challenge vs. performance.

Long story short: [BLAKE3](https://github.com/BLAKE3-team/BLAKE3) looks like a very promising hash, which is both better than SHA1 (so I'll be able to store *both* those SHA1-colliding PDF documents, which would *fail* if we were to merely *fix* the current SHA1 fingerprint encoding in Qiqqa) *plus* it's expected to be *faster* :yay:.


## Okay, that's the easy bit then. How do we remain backwards compatible or migrating forward?

Except for the colliding PDFs, the easy answer to that one would be to maintain a separate 1:1 lookup table in SQlite where old SHA1-B0RK fingerprints are mapped to new BLAKE3 hashes.

It would double the fingerprinting work as long as we want to stay backwards compat as then we'ld have to calculate both BLAKE3 and SHA1-B0RK, but that's manageable. (text extraction tasks, etc. are more costly, so let's try to see this *relatively*: fingerprinting * 2 is not going to kill the app.


## Any fears about 'cracking the hash' in the future? Other doubts & worries?

Not much. I'm not using the hash for pure cryptographic means, so a few odds and ends are not going to kill us. The fact that there exist real PDFs out there which collide on MD5 and SHA1 have made those hashes obsolete, but otherwise I believe we'll be doing fine with a reduced-strength, fast BLAKE3 for the foreseeable future.

Again, do realize we're not doing *crypto*/*security* here: the fingerprinting is also not employed in a *legal* setting where I need to unambiguously, beyond the shadow of doubt, *prrove* that my fingerprint identifies document X.
The fact that I'm paranoid enough already (perfectionist?) means I'll tend to pick a fingerprint which has great expectations that way anyway and so far, BLAKE3 is looking great.

### Totally unrelated but still *related* is another matter: *near-identical* PDFs cluttering the library

As you asked about my doubt & worries, here's one: I discovered last week that https://www.researchgate.net/ does something nasty (from *my* perspective): it kind-of-*personalizes* your PDF download by editing its cover page: it brags about the number of downaloads and the day you downloaded the PDF, so downloading the same document *twice* (say, via Qiqqa sniffer) will get you two(2) **different fingerprints**, thus the document downloads will be filed as unique documents in Qiqqa!

Sites which encode the date when you downloaded the PDF into the PDF itself (watermark/overlay) cause the samee problem: multiple downloads cannot easily be identifies as duplicates through comparing their hashes!

That's therefor another **source of duplicates** for documents and increases the need for Qiqqa to incorporate better duplicate-detection logic (including 'probable cause of duplication' type identification, so you can observe & set *why* two documents are considered *duplicates*:

- watermark: time/otherwise-edited unique copy watermarks a la https://www.researchgate.net/
- website coversheet: xArchiv and various universities or other sources may present the same content, but with different coversheets. Again https://www.researchgate.net/ vs others.
- prepub vs. print: prepublication version vs print version. 
- journal, i.e. *which* print version: sometimes articles are published in multiple journals, each with their own formatting (and *editing*, sometimes). I'd say this type also includes "personal copies" as published by ohne of the authors.
- updates: xArchiv and a few others keep track of the *revisions*: v1, v2, v3, and so on of the published paper
- aniversary re-issues / reprints: some papers are reprinted after X years. Happens rarely, but they do exist. It happens more often when you include *books* in your library. *Editions* of books would, as far as I'm conceerned, fall under the 'revisions/updates' type.
- amalgam/extract: sometimes papers are published in two versions: a short and a long one. Some papers are also *incorporated* into other papers, not just referenced, but re-used/quoted almost entirely, while other content is added. This goes beyond a mere 'update'.
- *plagiaat*: we now get into the more shady corners: some papers are copied or rephrased by others to make the publishing count ends meet. Plagiarism.
- *retractions*: that's a special kind of update, not really a *duplicate* per se, but more an *overriding* relationship. Remember our beautiful doctor who invented the pro-vegan argument of more aggression in meat-eating peeople while waiting for his train. Then the bugger went on to write a book about his life, earning yet more kuddos. (I'm proud to be Dutch!)

and when we veer a little off from true duplicates that way, there's also:

- reviews of: reviews of a book or paper, either itself a paper or just a memo published in a journal or otherwise.
- meta-reviews: *Meta-reviews* are those where the "current state of art" is discussed and multiple papers are reviewed together to report on where we are right now as a group.
- answers to: memos and letters to the editor in answer to paper X or memo Y, that sort of thing mostly.
- replication reports (where B attempted to replicate A. Sometimes successful, sometimes not. Cold fusion by loose wiring, anyone?)
- bundling of (I have several PDFs which happen to contain multiple papers: that would be a *book* itself, but having that paper X in there would make it a duplicate of that paper (amalgam or otherwise)

Now I hope some day we'll be able to identify most, if not all, of those "duplicates" and "other relations" automatically...

Meanwhile, let's keep our fingerprint nicely unique and bemoan, yet accept/live with, the ways researchgate et al are thwarting our efforts (mostly *unintentionally*).






## References

- https://crypto.stackexchange.com/questions/26336/sha-512-faster-than-sha-256
- https://en.wikipedia.org/wiki/Birthday_attack
- https://eprint.iacr.org/2019/1492.pdf
- https://en.wikipedia.org/wiki/Birthday_problem#Probability_table
- https://stackoverflow.com/questions/62664761/probability-of-hash-collision
- https://en.wikipedia.org/wiki/BLAKE_(hash_function)



