# Module 01: Curated Resources

A short, opinionated reading list. Annotated so you can choose what to read first.

We deliberately keep this list small. There is no shortage of platform-engineering content on the internet; the problem is signal-to-noise. Everything below has been read by the curriculum authors and is worth your time. Read in the order listed for the smoothest journey, or jump around based on your gaps.

> **A note on link rot.** Some URLs below may have moved or paywalled since the curriculum was last refreshed. If a link is dead, the title is searchable; the artifacts are widely-mirrored.

---

## Required (Read all four)

These are referenced throughout the lectures and exercises. You should read them at least once.

1. **["Meet Michelangelo: Uber's Machine Learning Platform"](https://www.uber.com/blog/michelangelo-machine-learning-platform/)** — Uber Engineering Blog, 2017.
   - *Why read it:* The canonical industry case study for an ML platform. Exercise 05 builds on this.
   - *What to look for:* The component breakdown (manage data → train → evaluate → deploy → predict → monitor). Note what the post does *not* describe (multi-tenancy, team structure, migration).
   - *Reading time:* 45 min.

2. **["Open-Sourcing Metaflow: A Human-Centric Framework for Data Science"](https://netflixtechblog.com/open-sourcing-metaflow-a-human-centric-framework-for-data-science-fa72e04a5d9)** — Netflix Tech Blog, 2019. Pair with the [Metaflow documentation](https://docs.metaflow.org/).
   - *Why read it:* The counterpoint to Michelangelo — SDK as platform vs services as platform.
   - *What to look for:* The "human-centric" framing. How does Metaflow get away with not building a serving layer or a feature store?
   - *Reading time:* 30 min for the post; another 30 min for the docs intro.

3. **["Platform Engineering"](https://martinfowler.com/articles/platform-engineering.html)** — Camille Fournier, on Martin Fowler's site.
   - *Why read it:* The definitive piece on the "platform-as-product" framing.
   - *What to look for:* The platform team's job description. The dangers of treating the platform as a hand-tooled project rather than a product.
   - *Reading time:* 25 min.

4. **["Hidden Technical Debt in Machine Learning Systems"](https://papers.nips.cc/paper/2015/file/86df7dcfd896fcaf2674f757a2463eba-Paper.pdf)** — Sculley et al., NeurIPS 2015.
   - *Why read it:* The original "5% ML, 95% glue code" paper. The arguments are timeless.
   - *What to look for:* The categories of debt (boundary erosion, data dependencies, configuration debt). Every category is something a platform should help mitigate.
   - *Reading time:* 30 min.

---

## Strongly Recommended

These deepen specific topics from Module 01.

5. **["A Philosophy of Software Design"](https://web.stanford.edu/~ouster/cgi-bin/book.php)** by John Ousterhout.
   - *Why read it:* The book is short and dense, focused on abstraction design. The single chapter "Modules Should Be Deep" is the best treatment of abstraction-design tradeoffs you'll read.
   - *What to look for:* The "deep modules" thesis, and the consequences for API design.
   - *Reading time:* 5-6 hours for the whole book; 30 min for the "Deep Modules" chapter alone.

6. **["Team Topologies"](https://teamtopologies.com/)** by Skelton & Pais.
   - *Why read it:* The taxonomy of team structures includes "platform teams" as a first-class category. The book gives you vocabulary for the team-structure decisions in Lecture 05.
   - *What to look for:* The four team types, the three interaction modes, and the chapter on platform teams specifically.
   - *Reading time:* 4-5 hours; the platform-team chapter is ~40 min.

7. **[Google API Design Guide](https://cloud.google.com/apis/design)** — Google's internal API design conventions, made public.
   - *Why read it:* When you next argue about API design with a colleague, you both want to be quoting the same source. This is a good shared source.
   - *What to look for:* Standard methods, resource naming, versioning. Treat as a *reference*; don't read end-to-end.
   - *Reading time:* 1 hour for a careful skim.

8. **[Kubernetes API Conventions](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/api-conventions.md)** — the conventions document that governs the Kubernetes API itself.
   - *Why read it:* Kubernetes' API is one of the largest, most-used APIs in modern infrastructure. The conventions reflect hard-won lessons.
   - *What to look for:* Object versioning, optional vs required, naming.
   - *Reading time:* 1.5 hours, dense.

9. **["Kubernetes documentation: Multi-Tenancy"](https://kubernetes.io/docs/concepts/security/multi-tenancy/)** — official Kubernetes docs.
   - *Why read it:* The official treatment of soft vs hard multi-tenancy and the tools Kubernetes provides for each.
   - *Reading time:* 45 min.

---

## Tool-Specific (skim as needed)

These are tool-specific. Don't try to read all of them now; come back when you need each.

10. **[MLflow documentation](https://mlflow.org/docs/latest/index.html).** The reference for experiment tracking and model registry. Module 07 will cover this in depth.

11. **[Feast documentation](https://docs.feast.dev/).** The reference for the canonical open-source feature store. Module 06 will dig in.

12. **[Kueue documentation](https://kueue.sigs.k8s.io/).** Kubernetes-native job queueing and fair sharing — relevant to the quota and scheduling work in Exercise 03 and Module 04.

13. **[Argo Workflows documentation](https://argoproj.github.io/argo-workflows/).** Workflow orchestration on Kubernetes; one of the most common ML pipeline tools. Module 04 / Module 05 territory.

14. **[KServe documentation](https://kserve.github.io/website/).** Kubernetes-native model serving. Module 08.

15. **[OpenCost](https://www.opencost.io/) / [Kubecost](https://www.kubecost.com/).** Open standard for Kubernetes cost allocation; useful for the cost-allocation discussion in Lecture 03 and Exercise 03.

16. **[OpenAPI Specification](https://swagger.io/specification/).** The spec itself; reference for Exercise 01.

---

## Going Deeper

For the genuinely curious. Skip on first pass; return as your interest narrows.

17. **["Designing Data-Intensive Applications"](https://dataintensive.net/)** by Martin Kleppmann.
    - The standard text on distributed-systems concerns relevant to platforms: consistency, replication, partition tolerance. Not platform-specific but foundational.

18. **["Building Microservices"](https://samnewman.io/books/building_microservices_2nd_edition/)** by Sam Newman (2nd ed.).
    - When you eventually need to break the monolith, this is the book.

19. **["Hidden Technical Debt" follow-on: "Machine Learning: The High-Interest Credit Card of Technical Debt"](https://research.google/pubs/pub43146/)** — Sculley et al.
    - A precursor / companion to the NeurIPS paper. Shorter, more polemical, very quotable.

20. **["The Twelve-Factor App"](https://12factor.net/).**
    - Heroku-era foundational text. Not ML-specific. The opinions about how applications should behave on a platform are still mostly correct.

21. **["The Hardest Part Is Selling Your Platform"](https://www.honeycomb.io/blog/hardest-part-selling-platform)** — Honeycomb blog, on the difficulty of internal platform adoption.
    - Practical, candid take on the Phase-1-to-Phase-2 adoption challenges from Lecture 02.

22. **["Building a Modern Data Platform"](https://www.databricks.com/blog/building-modern-data-platform)** — for the data-platform angle. ML platforms and data platforms are siblings; understanding both is useful.

23. **["What is a Feature Store"](https://www.tecton.ai/blog/what-is-a-feature-store/)** — Tecton's overview. Module 06 territory but the conceptual framing is useful here.

24. **CNCF Landscape: [Machine Learning](https://landscape.cncf.io/card-mode?category=machine-learning) and [Platform](https://landscape.cncf.io/card-mode?category=platform).** Browse to see what tools fit where in the reference topology. Don't try to memorize.

25. **["Production Machine Learning Pipelines"](https://www.oreilly.com/library/view/machine-learning-design/9781098115777/)** by Lakshmanan, Robinson, and Munn — "ML Design Patterns" (O'Reilly).
    - Patterns-style book; useful as a reference once you have specific problems.

---

## Talks Worth Watching

Conference talks complement reading. These are good first picks.

26. **["Platform Engineering at Spotify"](https://www.youtube.com/results?search_query=spotify+backstage+platform+engineering)** — search for recent talks; Spotify's Backstage team has spoken at many conferences. Get a sense of what a mature platform org looks like.

27. **["MLOps World" talks**](https://mlopsworld.com/) — annual conference; recordings often available. Quality varies; cherry-pick.

28. **["KubeCon ML Track" talks**](https://www.cncf.io/kubecon-cloudnativecon-events/) — recordings on YouTube. ML-on-Kubernetes specific.

29. **Camille Fournier's talks** — search YouTube. She has given multiple platform-engineering talks; any of them are worthwhile.

---

## Community / Newsletters

Stay in the loop without drowning.

30. **[Platform Engineering Slack](https://platformengineering.org/talks-library)** — community of platform engineers. Lurk; ask occasionally.

31. **[Internal Developer Platform](https://internaldeveloperplatform.org/)** — a community resource that classifies platforms and tools.

32. **["The Pragmatic Engineer" newsletter](https://newsletter.pragmaticengineer.com/)** — Gergely Orosz's newsletter often covers platform-engineering topics from a hiring / org perspective.

33. **[MLOps Community](https://mlops.community/)** — slack/podcast/articles. Variable quality; gems exist.

---

## What to Skip

To save you time, here is what *not* to start with:

- **Vendor whitepapers from major cloud providers about their own ML services.** They are marketing artifacts. Useful once you've internalized the principles; misleading as a first introduction.
- **Generic "10 best practices for X" listicles.** Almost universally low-signal.
- **Older (pre-2018) MLOps content.** The field has changed enough that earlier writeups are mostly of historical interest.
- **Twitter/X threads on "the right way to build an ML platform."** Sometimes brilliant, more often opinionated without evidence. Use as discussion starters, not as references.

---

## How to Use This List

A reasonable trajectory:

- **Module 01 (now):** Read items 1-4. Skim item 5's "Deep Modules" chapter. That's 2-3 hours of reading.
- **Module 02-03:** Read items 6-9.
- **Modules 04-08:** Read the relevant tool-specific items (10-16) as you encounter the tools.
- **Modules 09-10:** Read items 17-21.
- **As your career progresses:** Watch talks (26-29), join communities (30-33), read deeply (17-25).

You do not need to read everything to be a good platform engineer. You do need to read *something*, periodically, with intent.

---

## Contributing

If you find a great resource that should be on this list, open a PR in the curriculum repo with the resource and a 2-3 sentence annotation about why it belongs.

If a link is dead, file an issue with the current best URL (or a search term) and we'll update the list.

---

Next: Take the quiz (`quizzes/module-001-quiz.md`) if you haven't, and then advance to Module 02 (forthcoming): API Design for ML Platforms.
