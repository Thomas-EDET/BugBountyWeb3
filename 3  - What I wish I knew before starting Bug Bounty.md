# What I've Learned About the Bug Bounty World (So Far)

Over the past few months, I’ve been diving deep into the Web3 bug bounty space. Coming from six years of security auditing, I had certain expectations — but what I discovered was a bit more nuanced.

Two recent experiences in particular made me reflect on how bug bounty platforms operate in practice:

- Browsing through the leaderboards on some platforms
- Reading a detailed postmortem from Chainlink on recent attacks

## The Landscape Behind the Leaderboards

While browsing the Cantina leaderboard, I noticed a top researcher, _zigtur_, listed as part of Spearbit. Curious, I followed the trail — from zigtur’s profile to Spearbit’s website, then back to Cantina, where I found that **Spearbit Services** powers Cantina's audit team.

That made something click.

See:

![Screenshot from 2025-06-25 09-58-50](https://github.com/user-attachments/assets/1827cbcb-7de4-4b87-91cc-952038c24845)


![Screenshot from 2025-06-25 10-00-09](https://github.com/user-attachments/assets/801bbf8c-e1a2-420d-a45b-21b0993e04f5)


![Screenshot from 2025-06-25 10-03-59](https://github.com/user-attachments/assets/be1b68a6-7b20-4b98-885c-fb65669215e1)


There seems to be an informal **three-tier structure** among security researchers on many bug bounty platforms:

- **Tier 1**: Platform-backed audit teams (often internal or affiliated)
- **Tier 2**: Recognized researchers or trusted community members
- **Tier 3**: Newcomers trying to break in

I’m currently in Tier 3 — and what I’ve seen is that moving up isn’t purely meritocratic.

Even with a solid auditing background, my first valid vulnerability took **four submissions** just to be acknowledged. Several were closed with little or no explanation:

- _“AI scanners are forbidden”_ — although I included a custom PoC in Solidity
- _“Out of scope”_ — when the target was explicitly listed
- _“Issue fixed via documentation, finding invalid”_ — after 36 hours of focused review

And these aren’t isolated incidents — they’ve occurred across **multiple platforms**.

These barriers have unintended consequences.

---

### Ethics, Incentives, and Friction

According to a [HackerOne study](https://portswigger.net/daily-swig/bug-bounty-earnings-soar-but-63-of-ethical-hackers-have-withheld-security-flaws-study), **63% of ethical hackers have at some point withheld vulnerabilities**, and **1 in 5** have bent the rules.


In Chainlink’s [postmortem on a real-world exploit](https://drive.google.com/file/d/1G3obulCm_Z88ajsMc63oSHRIssLsXUhI/view), it’s mentioned that due to frustration with triage or fear of underpayment, some researchers may **"launder funds before disclosure"** or switch to private sale — a harsh reality.
![Screenshot from 2025-06-25 10-43-39](https://github.com/user-attachments/assets/23bcdef3-25bb-414e-8e79-0b3b88058c68)


Jason Haddix also spoke about this in one of his videos: that some platforms **mine researcher reports for techniques to build internal AI tooling or SAST products**, often without direct benefit or acknowledgment to the original researcher. That raises serious questions of transparency and consent.
![Screenshot from 2025-06-25 12-00-21](https://github.com/user-attachments/assets/685955d0-c976-4ddf-bfab-df237ec0a1b1)
Source:https://www.youtube.com/watch?v=6SNy0u6pYOc&t=236s

---

### A Few Reflections

I’m not writing this to criticize any individual platform or team — I still believe bug bounties are one of the most powerful tools we have to secure decentralized systems.

But we need to talk openly about:

- The opacity of triage and trust-building processes
- How newcomers are evaluated (or filtered out)
- Whether the current incentive structures unintentionally create frustration, burnout, or worse
- The use of researcher IP (e.g. custom exploit methods) without clear compensation or recognition

If we want bug bounties to be a sustainable model — especially in Web3 — we need to evolve these systems with transparency, fairness, and accountability in mind.


If you’ve faced similar experiences or found good ways to navigate these issues, I’d love to hear about it.

`Still actively participating in bug bounty programs and learning every day — just sharing honest reflections so others don't feel alone in the journey.` :heart:
