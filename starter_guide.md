## E-Voting Public Intrusion Test Starter Guide

#### Introduction
We are setuid(0), a group of security researchers/whitehats who are spending way too much participating at this public intrusion test.

With this guide we want to give people a quick start, so they can build their own opinion and have a smooth start for researching.

#### What does this guide cover?

It covers everything except complex mathematics and cryptography. For this part we are no specialists but we will quickly give you references. In the source code folder you can find the following files:

* Scytl sVote Audit with Control Components.pdf
* Scytl sVote Protocol Specifications.pdf

Here you can find the mathematical proofs for the math/crypto used:

* https://www.post.ch/-/media/post/evoting/dokumente/complete-verifiability-security-proof-report.pdf?la=en&vs=1
* https://www.post.ch/-/media/post/evoting/dokumente/privacy-security-proof-report.pdf?la=en&vs=1
* https://www.post.ch/-/media/post/evoting/dokumente/complete-verifiability-formal-proof-report.pdf?la=en&vs=1
* https://www.post.ch/-/media/post/evoting/dokumente/privacy-formal-proof-report.pdf?la=en&vs=1

These are our experiences after spending several dozens of hours at the PIT. 
Feel free to correct mistakes or add your realisations. 

---

### First Steps

- Get access to the code repository 
- Register at onlinevote-pit.ch
- Hereby you accept all terms of service/code of conduct

Afterwards you should inform yourself about the scope - this is essential in this document.

#### Public Information

First off, one needs to realize that this is not your "average bug bounty program". It is the swiss evoting system, therefore you should read publicly available information to get a broader picture, about the aim of the bug bounty, the history and so on.

The fact that there were already penetration tests conducted before the PIT, there are 3k+ participants and the source code got already released 2 weeks before the PIT, should tell you that you can safe time by jumping right into code analysis than just tinkering around with your scanners.

#### Infrastructural Analysis

So first off there are two domains in scope:
* pit.evoting-test.ch
    * Open Ports: 80, 443
* pit-admin.evoting-test.ch
    * Open Ports: 80, 443

Even with the source code, we found no way to make any communication to pit-admin.evoting-test.ch. This is the online SDM, we will talk later about this in detail. We just got 403, 403 and even more 403's!

Another important thing to look at is, they clearly stated that there are no "advanced security measurements" deployed for the pit. 

We don't suspect that there are advanced firewall rules. But sometimes we expected a certain endpoint to be accessible from outside because the REST endpoint clearly had no filter to block it and it was exposed the same way the accessible ones are in the same class. But somehow, we were given another 403.

**In conclusion:** Go for pit.evoting-test.ch

#### The scope

The scope is your best comerade to actually have success. We had big issues with the scope in the beginning of our analysis, as you can see from our release #1 publications.


The scope on the website is a bit confusing.
There are two components which are called secure-data-manager or short sdm (secure in title is a big red flag xD). 
Only one of them is in scope. There's the online-sdm which is in scope (but unaccessible due to 403's) and the offline sdm pre/post election which runs on the cantonal premises in a "high security" airgapped system.


The PRE- and POST voting configurations are out of scope, we will explain this further below.


#### System Analysis 

This is a quick overview of the systems in a very abstract manner:

* Voting Portal 
    * Frontend Application for voters (Not included in the source code)
* Voting Channel
    * Voting Backend
* Mixnet
    * For mixing the votes
* Secure Data Manager
    * Pre/Post voting configuration
    * There is a "online SDM" which connects with the voting channel


With these terms cleared up we want to jump in the source code.

---

### Source Code Analysis

First off, forget about building and running the source code. This takes up way too much time, and nobody of the PIT participants has been able to run elections with the code.

Now the source code is huge, so first we want to make it easier for you by defining low-key elements (not application entrypoints), in the folder evoting-solution-master/source-code:

* scytl-math
    * Implements custom math stuff, even with C bigint stuff.
* scytl-cryptolib
    * Does what it says, provides a cryptographic library.
* scytl-secure-logger
    * This implements a huge logger, basically everywhere in the whole program LOG.LOG() is called, which generates encrypted logs on the system.
* maven-generic-config & ov-maven-dependencies
    * Ignore, just some build stuff.
* online-voting-logging
    * Logging stuff, also puts data together for splunk.
* online-voting-mixing
    * Does the whole vote mixing.

All the parts above are interesting for a long-term analysis.

Lets get to the clientside stuff, we think that the clientside stuff is not that important in the context of the PIT, because of the defined scope:

* online-voting-client-library
    * This is the javascript client library which implements all the backend functions for the frontend "Voting Platform" (ov-api.js).
    * Example functions: castVote(), parseBallot(), translateBallot()
    * Documentation in: online-voting-client-library/doc/api-guide.md

Now these are the parts which actually the PIT "begins", because here are the application entrypoints:

* online-voting-secure-data-manager
    * This is the "Secure Data Manager".
    * Partially in scope, but basically completely useless in the context of the PIT, because (in our experience) pit-admin.evoting-test.ch is unaccessable. (403 party)
    * We already reported some funny stuff in our #1 release, but hey, its an airgapped system and nobody cares :D 
    * We decided to stop our analysis of the SDM completely because of the scope.

* online-voting-channel
    * This is the most important part of the whole PIT.
    * It is the backend of the "Voting Platform", the client-library communicates with the voting-channel to do the actual operations.
    * ov-api-gateway (ag-ws-rest) is used as a proxy class to do retrofit2 requests to other rest endpoints.

**In conclusion:**
Go for the voting channel.


### Voting Channel Analysis

So here the real deal begins, this is the most valuable part of this guide.
During the PIT, get yourself some voting cards and use them on the evoting system. If you look at your burp/zap/whatever/proxy setup, you will see ~6 calls being done to the REST endpoint "ag-ws-rest".

Welcome at the "scope" of this PIT, aka. 9 rest endpoints.
**VERY IMPORTANT:** We haven't checked every one of the hundreds of endpoints, but we got a lot of 403 responses, so we derive from the voting-api raml file.

Ok, so what's going on here?


#### Ov-Api-Gateway

This module (the ag-ws-rest endpoint) serves as a rest endpoint which acts as a proxy between the client library and the different endpoints.

Now because we are trying to crack down further entrypoints for analysis, we have to think about what is actually accessible.

It is pretty clear that the Ov-Api-Gateway is the only part which is being accessable from outside during the PIT, because every other endpoint implements a SignedRequestFilter, which will check for a certain certificate. But the other parts anyway didn't seem to be accessible from outside.


So let's go to the next question, is everything accessible inside the Ov-Api-Gateway? 
Quick answer: NOPE
We only found the endpoints in the "voting" package to be accessible from the outside. We believe that a firewall rule blocks the other functionalities.

These are the 9 available rest endpoints:


| Package | File | Function | Implementation & Client |
| -------- | -------- | -------- | -------- |
| voting     | AuthenticationTokenResource | getAuthenticationToken()| ov-voting-workflow |
| voting     | CastCodeResource | getCastCodeMessage()| ov-voting-workflow |
| voting     | ChoiceCodeResource | getChoiceCodes()| ov-voting-workflow |
| voting     | ConfirmationMessageResource | validateConfirmationMessage()| ov-voting-workflow |
| voting     | CredentialInformationResource | getAuthenticationInformation()| ov-voting-workflow |
| voting     | ExtendedAuthenticationResource | getEncryptedStartVotingKey()| ov-extended-authentication |
| voting     | ExtendedAuthenticationResource | updateExtendedAuthData()| ov-extended-authentication |
| voting     | ReceiptResource | getReceiptByVotingCardId()| ov-voting-workflow |
| voting     | VoteResource | validateVoteAndStore()| ov-voting-workflow |


So you should try to focus on these endpoints.
First you can go into the ov-api-gateway to see where the function is and then trace it to ov-voting-workflow or ov-extended-voting.

Altogether those are the 9 rest endpoints which are basically "the entrypoint" to the PIT.

#### What next?
Well now you can analyze in detail to find bugs, we hope this guide helped you out a little bit and gave you a quick start over everything.


## Final Words

As we said we don't want to discuss #EVoting as a concept. We state again that we are neutral wether e-voting is a good idea or not. Yet we wan't to give our two cents to some of the "misconceptions" found in the #EVoting discussions.

```
Stop the bad PR! If you do bad PR there will be no Bug Bounties in Switzerland!
```
We think this argumentation is flawed on many levels. This is NOT a common bug bounty. Not every bug bounty decides about democracy, politics and public opinion.
We don't want to get political but one can agree that it's critical infrastructure at least.
``` PoC || GTFO ```
There are these "experts" who are spamming this all over the place. A lot of the time we could theoretically provide proof of concepts, but were unable because there was no code to be ran! Also, please, get real, what is a proof of concept worth if most part is out of scope. Yes, one should proof of concept, we agree, but to shout this around wildy without contributing anything useful is not helping anybody.

Just if you think about it for a few minutes, in normal conversations, do you have to directly proof everything? For example "this bridge could break down because of people hanging love locks on the side", "PoC || GTFO!!! Show me how this bridge breaks down by providing thousands of love locks"! 
First think about the background why somebody can't provide a PoC and yes, you will be surprised, it sometimes doesn't means that the person is a noob.


## Something to think about

There are people saying the following:
```The PIT is a farce/PR stunt```

Now first off it is nothing wrong to host a bug bounty with minimal scope. Basically nobody cares, because everyone sees that and waves goodbye after a few hours of analysis and moves to another project. 

But if a whole country looks at a "big hacker test" which they say "isn't about security", but they want to "learn from hackers", it gets pretty confusing really quick. The price money was 250k. 100k is gone already, probably because of the organisational costs.

So in the price pool there is 150k, lets keep that in mind.

Now this test will fundamentaly shape the views of people about IT-security, there might be swiss politicians which point to this test in the future and say "here you see, we are safe". 

Also in our opinion, the WHOLE system should be tested. If we look at the PIT currently, we only have access to a *fraction* of it. We see the the system as an onion. If you can only check the outer layers, the core can still be rotten. 

If you would, for example, be one hop in the internal network where the voting channel is running, these firewall rules might be gone and you could potentially access the unprotected endpoints of ov-api-gateway.

```
But you noobs, it is security best-practice to put a firewall in front of such an application!
```

Yes, but the application should also be constructed in a way, that if someone changes the firewall rules it still would operate securly. It is simply not best practice to have unprotected rest endpoints, just because they are not reachable.

In the end, from our point of view, it was clear that the system is vulnerable by design. This was shown by our first release, which shows that certain parts don't have any security requirements at all, since they run airgapped.

We spent our time to fiddle around with the REST endpoints, trying to squeeze the lemon as hard as we could to get some lemonade.

There would be so much more to be tested and investigated, but we decided to stop at this point. The scope is way too small and we don't want to provide freebie patches while somebody gets 150k without any real efford.

The last thing what we want to say is we are just young hackers having fun. All the "professionals" are just missing out because "they have no time", right?

But even though, we were some of the few that published actual vulnerabilities while almost everyone else was just badmouthing the code without any security impact. Keep that in mind.


***micdrop***

```
                    .     .         __    `   _     __
                         ________  / /___  __(_)___/ /
                  *     / ___/ _ \/ __/ / / / / __  /    *
                       (__  )  __/ /_/ /_/ / / /_/ /
                      /____/\___/\__/\__,_/_/\__,_/ (0)
                  `                           .         `

```
