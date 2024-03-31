---
layout: post
title: Fault Injection and the Supply Chain
---

## NullCon Berlin 2024!

To provide some context, this was a CFP that was submitted to, accepted, and presented at NullCon Berlin 2024 by myself (Nolen Johnson) and Jan Altensen.

## Presentation

[![](https://markdown-videos-api.jorgenkh.no/youtube/RrJQ0KE7klc)](https://youtu.be/RrJQ0KE7klc)

## Abstract

This paper investigates the critical intersection of fault injection techniques and the supply chain, unveiling a previously overlooked threat vector in both consumer and enterprise devices.

Focusing on the ubiquitous Chromecast with Google TV, we demonstrate that malicious actors can exploit vulnerabilities within u-boot related to the handling of storage devices through fault injection, introducing malware into the supply chain undetected.

This research exposes a new dimension of security risks associated with the manufacturing and distribution processes of widely-used consumer electronics.

Our primary focus centers on the Chromecast (and other Amlogic devices) eMMC fault injection, privilege escalation, and persistence methods, revealing a potential avenue for adversaries to compromise the integrity of these devices and many other device utilizing Amlogic chipsets.

By manipulating the device physically a single time, attackers can clandestinely inject persistent malware/spyware, thereby compromising user privacy, data security, and the overall functionality of aected devices. This study underscores the necessity for enhanced security measures throughout the supply chain to mitigate the risk of fault injection attacks, as well as more careful review of edge cases by development teams. As the Chromecast represents just one example in a broader ecosystem of connected systems. Our findings emphasize the urgency of proactive measures to fortify the security of embedded systems and safeguard against ever evolving threats.

In this talk, we demonstrate a full persistent secure-boot bypass for the [Chromecast with Google TV](https://oddsolutions.github.io/Chromecast-with-Google-TV-1080P-Secure-Boot-Bypass/), as well as an upcoming persistent secure-boot bypass for the Pixel Tablet dock.

### References
* [CVE-2023-48424](https://source.android.com/docs/security/bulletin/chromecast/2023-12-01#amlogic) - eMMC fault injection (attributed to both our team and the HexTree.io team)
* [CVE-2023-48425](https://source.android.com/docs/security/bulletin/chromecast/2023-12-01#amlogic) - AVB (upgradestep) vulnerability in u-boot
* [CVE-2023-6181](https://source.android.com/docs/security/bulletin/chromecast/2023-12-01#amlogic) - Proper whitelisting of BCB commands passed by the user-space
* TBA - Pixel Tablet Dock Vulnerability 1
* TBA - Pixel Tablet Dock Vulnerability 2

### Credits
* Myself - Creating/Architecting the paper submission, presenting
* Jan Altensen - Assisting in submission development, presenting
* Ray Volpe - Credited for working with us on the Chromecast exploit chain
* Chris Walcutt - Putting up with my repeated practices, providing fantastic feedback, and being a fantastic mentor.
* Philip Valvo - Providing honest and helpful feedback, helped retool the flow of the presentation. Also being my favorite coworker of all time, and best friend. [#PourOneOutForOurFallenBrother](https://www.youtube.com/watch?v=F8MlRGvri-c)
