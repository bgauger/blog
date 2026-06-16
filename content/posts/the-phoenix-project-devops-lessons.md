---
title: "The Phoenix Project: DevOps Lessons That Changed How I Run My Homelab"
date: 2026-01-06
draft: false
tags:
  - devops
  - books
  - infrastructure
  - automation
  - continuous-improvement
---

As a DevOps engineer, I thought I understood the principles behind my work. Then I read *The Phoenix Project* by Gene Kim, and suddenly every production incident, every failed deployment, and every 2 AM page made perfect sense. This book doesn't just explain DevOps—it shows you exactly why traditional IT operations fail and how to fix them.

<!--more-->

## What Is The Phoenix Project?

*The Phoenix Project: A Novel About IT, DevOps, and Helping Your Business Win* is a business novel that follows Bill Palmer, an IT manager suddenly promoted to VP of IT Operations at Parts Unlimited. The company is bleeding money, their critical Phoenix Project is years overdue, and IT is blamed for everything.

I've lived this story. Maybe not as dramatically as Bill, but I've been in the war rooms, the postmortem meetings, and the "all hands on deck" Teams channels when production goes down. Reading this book felt like watching my own career play out through someone else's disasters.

The book introduces **The Three Ways** of DevOps through a narrative format that makes complex concepts instantly relatable. It's not a technical manual—it's a story that makes you *feel* the pain of poor practices and the relief of getting it right.

## The Three Ways of DevOps

The book's core framework consists of three principles that transform how organizations (or individuals) approach IT operations:

### The First Way: Flow

**Principle:** Optimize the flow of work from development to operations to the customer.

In the book, Bill discovers that work-in-progress (WIP) kills flow. Parts Unlimited had hundreds of projects in flight, all stuck in various stages of completion. Nothing ever finished.

**At work:** I've seen teams with 15 PRs open, 8 feature branches, and 3 major initiatives all "in progress." Developers context-switch between projects while ops scrambles to keep up. Release cycles stretch from weeks to months because nothing is ever truly complete.

**The fix:** Limit WIP ruthlessly. We implemented WIP limits on our kanban board—no more than 3 items in "In Progress" at once. Finish what you start. Close the PR before starting the next feature.

**In my homelab:** Same principle applies. I had Portainer partially deployed, Terraform configs that didn't match reality, and manual configurations scattered across servers. Now I finish one automation project before starting the next.

### The Second Way: Feedback

**Principle:** Create fast feedback loops to detect and fix problems quickly.

In the book, Parts Unlimited deployed code without knowing if it worked until customers complained. By the time problems were discovered, finding the root cause was nearly impossible.

**At work:** We've all been there—deploy on Friday, discover the bug on Monday when users can't log in. No health checks, no smoke tests, no monitoring that actually alerts on business impact. By the time you're paged, the blast radius is massive.

**The fix:** We built comprehensive feedback loops:
- CI/CD pipelines with automated testing
- Canary deployments to catch issues before full rollout
- Synthetic monitoring that simulates user journeys
- Observability (not just monitoring) with distributed tracing
- Blameless postmortems to learn from incidents

**In my homelab:** I applied the same thinking. Deployed Portainer for centralized monitoring, added health checks to Docker Compose files, set up log aggregation. Now problems surface in minutes, not when I try to watch a movie and Jellyfin is down.

### The Third Way: Continuous Experimentation and Learning

**Principle:** Create a culture of experimentation, taking risks, and learning from failure.

In the book, the IT team was so afraid of breaking production that they never tried anything new. Ironically, this fear caused more outages because they couldn't evolve fast enough.

**At work:** I've worked at companies where the culture was "don't break production at any cost." Sounds good, right? Wrong. It meant no one wanted to upgrade dependencies, refactor legacy code, or try new approaches. We avoided small, manageable risks and accumulated massive technical debt that eventually caused catastrophic failures.

**The fix:** We shifted to a culture of safe experimentation:
- Feature flags to test in production without risk
- Chaos engineering to build resilience
- "20% time" for exploring new tools and approaches
- Blameless postmortems that celebrate learning
- Infrastructure-as-code so environments are disposable

**In my homelab:** Same mindset. With Terraform and Ansible, I can spin up test environments in minutes. I broke my test server three times perfecting cloud-init automation—didn't matter because I could rebuild it in 10 minutes. That confidence to experiment transferred back to work.

## The Four Types of Work

One of the book's most powerful concepts is categorizing work into four types:

1. **Business Projects** - New features, new services (e.g., deploying Jellyfin)
2. **Internal IT Projects** - Infrastructure improvements (e.g., migrating to Terraform)
3. **Changes** - All the modifications and updates (e.g., updating Docker images)
4. **Unplanned Work** - Firefighting and fixes (e.g., "why is the service down again?")

The insight: **Unplanned work kills everything else.**

Every time you're paged at 3 AM because a service crashed, that's unplanned work. Every emergency hotfix that bypasses your deployment pipeline is unplanned work. Every "quick manual change" that causes an outage is unplanned work.

Here's a perfect example from a previous role: Every year, we had to re-STIG our entire environment.**

STIGs (Security Technical Implementation Guidelines) had to be manually applied to every Windows server—hundreds of configuration checks per system. The process took **months** of manual work. Every. Single. Year.

The solution? Automate it once with PowerShell and never do it manually again.

But we never "had time" to build that automation because we were too busy manually re-STIGing servers. This is the vicious cycle—unplanned and repetitive work consumes all available time, preventing you from eliminating the root cause.

**The vicious cycle:**
1. Too busy firefighting to automate
2. Because you don't automate, you have more fires
3. Leadership sees you're "busy" and adds more resources
4. More people create more unplanned work
5. Repeat until the entire org is firefighting

**The virtuous cycle:**
1. Invest time in automation (CI/CD, IaC, monitoring)
2. Fires decrease, freeing up time
3. Use that time to eliminate more toil
4. Eventually, your infrastructure is self-healing and your on-call rotation is quiet

## Key Lessons Applied to Professional DevOps

### 1. Make Work Visible

In the book, Bill creates a kanban board to visualize all IT work. Suddenly, the invisible becomes visible—and manageable.

**At work:** We implemented this with Jira boards, daily standups, and WIP limits. Management could finally *see* why the ops team was always "busy" but projects never shipped—we had 40 things in progress and nothing reaching "Done."

**At home:** Same principle. I use Kanboard to track homelab infrastructure changes. No more "I think I need to update that thing on that server."

### 2. Reduce Batch Size

Parts Unlimited tried deploying massive changes once a quarter. Every deployment was high-risk chaos.

**At work:** We batch changes into releases every few sprints. While not continuous deployment, it's still smaller, more manageable releases than massive quarterly deployments. Smaller batch sizes mean lower risk, faster feedback, and easier rollbacks when needed.

**At home:** Same approach. Incremental improvements. Update one service at a time. Deploy one automation playbook at a time. Small, safe, frequent changes.

### 3. Eliminate Constraints

In manufacturing (and IT), the Theory of Constraints says: identify your bottleneck and optimize it. Everything else is a distraction.

**At work:** Our bottleneck was a cybersecurity person who wanted to be involved in everything. Every change, no matter how small, required their approval. Their default answer? "It's already working, no need to change it." Progress ground to a halt because one person became a gate on all improvements.

**At home:** My bottleneck was manual SSH configuration. Every new service meant 30 minutes of setup. Ansible eliminated that constraint. Now deployments take 5 minutes and are reproducible.

### 4. Build Quality In

Don't inspect quality in at the end—build it into the process.

**At work:**
- Automated testing in CI/CD pipelines
- Infrastructure-as-code with Terraform
- Policy-as-code with Open Policy Agent
- Automated security scanning
- Gradual rollouts with automatic rollback

**At home:**
- Health checks in Docker Compose
- Idempotent Ansible playbooks (can run multiple times safely)
- Version control for all configs
- Testing in isolated environments before production

### 5. Create Telemetry

You can't improve what you can't measure.

**At work:**
- Prometheus + Grafana for infrastructure metrics
- ELK stack for log aggregation
- Distributed tracing with Jaeger
- SLO-based alerting (not just "is it down?")
- DORA metrics to track team performance

**At home:**
- Portainer dashboards for container health
- Basic log aggregation
- Uptime monitoring
- Git commit history showing what changed when

## The Book's Biggest Revelation

The most powerful moment in *The Phoenix Project* is when Bill realizes: **IT Operations is manufacturing.**

The same principles that transformed manufacturing in the 20th century (Lean, Theory of Constraints, Total Quality Management) apply directly to IT:

- **Work in progress kills throughput** - Finish what you start
- **Batch sizes matter** - Small, frequent changes are safer than big-bang deployments
- **Constraints determine capacity** - Find your bottleneck and fix it
- **Feedback loops prevent defects** - Catch problems early when they're cheap to fix
- **Continuous improvement compounds** - Small improvements add up exponentially

This reframe changed everything for me. My homelab isn't a collection of servers—it's a **production line** where I should optimize flow, minimize WIP, and build quality in.

## Practical Takeaways for DevOps Teams

If you haven't read *The Phoenix Project*, here's what to do next:

1. **Read the book (or listen on your commute)** - It's a novel, not a textbook. Engaging, practical, and you'll finish it in a few days.

2. **Audit your unplanned work** - Track how much time your team spends firefighting vs. improving. If it's more than 20%, you have a problem that requires systemic fixes, not more people.

3. **Make work visible** - Kanban boards, sprint planning, whatever works. You can't manage what you can't see.

4. **Identify your constraint** - Where's the bottleneck in your deployment pipeline? Manual approvals? Slow tests? Fragile deployments? Fix that *first*.

5. **Build fast feedback loops** - Automated testing, gradual rollouts, observability. Catch problems in seconds, not after customer complaints.

6. **Automate toil** - What does your team do manually every week? That's your automation target.

7. **Limit WIP** - No more than 3 major initiatives at once. Finish things. Ship them. Then start the next one.

8. **Create safe environments to experiment** - Infrastructure-as-code, ephemeral environments, feature flags. Make it safe to try new things.

9. **Measure what matters** - Track DORA metrics (deployment frequency, lead time, MTTR, change failure rate), not vanity metrics like lines of code.

## How This Changed My Approach to DevOps

**Before reading The Phoenix Project:**
- Saw DevOps as a collection of tools (Docker, Kubernetes, CI/CD)
- Focused on local optimizations (make *this* deployment faster)
- Accepted firefighting as "part of the job"
- Measured success by hours worked, not outcomes delivered
- Thought automation was "nice to have" when there's time

**After understanding the principles:**
- DevOps is about **flow, feedback, and continuous learning**—tools are just enablers
- Optimize the entire system, not individual components
- Firefighting is a symptom of broken processes, not an inevitable reality
- Measure success by lead time, deployment frequency, MTTR, and change failure rate
- Automation is the foundation—you *make* time by automating

**At work:**
- Deployment frequency: improved through better release planning
- Change failure rate: reduced through automated testing and smaller batches
- Mean time to recovery: significantly improved with better automation
- Unplanned work: dramatically reduced through infrastructure-as-code
- On-call stability: much more predictable with proper monitoring

**At home:**
- Infrastructure-as-code for complete reproducibility
- Test environments I can destroy and rebuild in minutes
- Monitoring catches issues before I notice
- Confidence to experiment without fear
- Time saved per week: 3-4 hours that used to go to firefighting

## My Journey: Seeing Both Sides of the DevOps Divide

Reading *The Phoenix Project* hit especially hard because I've lived both sides of the story—organizations that resist automation and those that embrace it.

### The Prior Company: Automation Without Culture

A few years ago, I worked on a contract where the client was migrating their on-premises infrastructure to AWS. It was a massive lift-and-shift operation—servers moved to the cloud, but processes stayed firmly rooted in the past.

My mentor on that project recommended we import everything into Terraform. He saw the same chaos I did—hundreds of EC2 instances, security groups, load balancers, all manually created through the AWS console with zero documentation. Like Erik teaching Bill in *The Phoenix Project*, he was trying to guide us toward better practices.

I took on the work: tedious reverse-engineering of infrastructure decisions made months earlier, bringing everything under infrastructure-as-code management.

But I saw the value. Once in Terraform, we could:
- Track changes in version control
- Review infrastructure modifications like code
- Rebuild environments consistently
- Understand dependencies between resources

Then I tackled the biggest time sink: the annual STIG process.

Remember that months-long manual effort I mentioned earlier? Every year, the team had to manually re-STIG the entire environment—hundreds of configuration checks per Windows server. I built PowerShell automation to do it all consistently and repeatably. What took months manually could be done in minutes with automation.

**Here's the problem:** Nobody cared.

The Terraform configs? Used initially, then abandoned as people made "quick changes" directly in the console. The STIG automation? Ignored in favor of manual processes "because that's how we've always done it."

When I moved off the project, I heard they stopped using all the automation I'd created. They went back to ClickOps, manual server configuration, and spending months every year re-STIGing servers by hand.

Reading *The Phoenix Project*, I finally understood why: **You can't fix culture with tools.**

My mentor had shown us the path, just like Erik guides Bill in the book. But guidance isn't enough when the organization resists change.

We had built automation, but the organization hadn't embraced the principles:
- No commitment to infrastructure-as-code (First Way - Flow)
- No feedback loops to show automation's value (Second Way - Feedback)
- No culture of learning or experimentation (Third Way - Continuous Learning)

The automation failed because the **culture** didn't support it. Bill Palmer faces this exact problem—even with Erik's mentorship, he has to fight organizational resistance at every step. Having a guide doesn't matter if the organization won't follow.

### The Current Company: DevOps as the Standard

Later, I joined a company where infrastructure-as-code and DevOps aren't just buzzwords—they're non-negotiable standards.

**Everything** is in code:
- Infrastructure provisioned with Terraform
- Configuration managed with Ansible
- Deployments automated through CI/CD pipelines
- Changes reviewed like software commits

**Nobody** makes manual changes in production. If you need something modified, you:
1. Update the code
2. Submit a pull request
3. Get it reviewed
4. Merge and let automation deploy it

The contrast was stunning. Same tools (Terraform, AWS, automation), completely different outcomes.

**At the prior company:**
- Infrastructure drift was constant
- Audits were stressful scrambles
- Deployments were manual and error-prone
- Knowledge lived in people's heads
- Bus factor was critical (what if someone leaves?)

**At the new company:**
- Infrastructure matches code (drift is detected and fixed)
- Compliance is automated and provable
- Deployments are reliable and frequent
- Knowledge lives in repositories
- Anyone can understand and modify infrastructure

**The difference?** Not the tools. Not the talent. Not the budget.

**The culture.**

One organization saw automation as "nice to have" and abandoned it when inconvenient. The other saw automation as foundational and refused to operate without it.

This is exactly what *The Phoenix Project* teaches: DevOps transformation isn't about adopting tools—it's about changing how organizations think about work, risk, and improvement.

### The Lesson

When I read about Parts Unlimited struggling to adopt change, I saw my prior company experience. When I read about Bill's eventual success building DevOps culture, I saw my current company.

**The book's core truth:** You can have all the right tools and still fail if the culture fights you. Conversely, a strong DevOps culture will succeed even with simpler tools because they understand the principles.

My Terraform code at the prior company could have saved hundreds of hours and prevented countless errors. But without organizational buy-in, it was just unused files in a repository.

At my current company, that same Terraform code would be:
- Maintained and improved over time
- Used daily by the entire team
- Integrated into deployment pipelines
- Treated as critical infrastructure

Same code. Different culture. Completely different results.

## Why Every DevOps Engineer Should Read This

This book resonates differently depending on your role:

- **DevOps/SRE engineers** will see their own production incidents reflected in Bill's disasters
- **Developers** will finally understand why "just deploy it" isn't simple and why ops teams push back
- **Engineering managers** will recognize how invisible work and technical debt compound
- **CTOs/VPs** will see how organizational dysfunction creates technical problems

The book's genius is showing that DevOps isn't about tools (Kubernetes, Docker, CI/CD)—it's about **principles** that guide how you use those tools effectively.

I've worked at companies that had all the latest tech—Kubernetes, GitOps, microservices—but still struggled with deployments because they ignored the Three Ways. I've also worked at companies with simpler tech stacks that shipped faster and more reliably because they understood flow, feedback, and continuous learning.

## Related Reading

*The Phoenix Project* is the first in a series. If it resonates, check out:

- **The Unicorn Project** - Companion novel from a developer's perspective
- **The DevOps Handbook** - Technical deep-dive into implementing these practices
- **Accelerate** - Research proving these principles work (with data)
- **The Goal** - The manufacturing book that inspired The Phoenix Project

## Final Thoughts

*The Phoenix Project* taught me that DevOps problems are rarely technical—they're almost always process and culture problems.

Early in my career, I thought DevOps success meant adopting the right tools. Kubernetes, Terraform, CI/CD pipelines, microservices. But I've seen teams with all the latest tech still struggle to deploy reliably. I've also seen teams with simpler stacks ship faster and more safely.

The difference wasn't the tools—it was understanding the principles:
- Optimize for flow, not individual efficiency
- Build fast feedback loops into everything
- Make it safe to experiment and learn from failure
- Eliminate toil through automation
- Make work visible and limit WIP

These principles apply whether you're managing a Fortune 500's infrastructure or your homelab at home. The scale changes, but the fundamentals remain the same.

If you're in DevOps, SRE, platform engineering, or any role managing complex systems, read this book. It codifies lessons that took me years of production incidents to learn the hard way.

---

**What I'm working on next:**
- Deploying legacy apps into the cloud with GitOps
- Expanding automated testing in our deployment pipelines
- Applying chaos engineering principles
- In my homelab: finishing centralized logging and automated backups

**Have you read The Phoenix Project? Did it change how you approach DevOps? What lessons hit hardest for you?**

## Resources

- [The Phoenix Project on Amazon](https://www.amazon.com/Phoenix-Project-DevOps-Helping-Business/dp/1942788290)
- [IT Revolution (Publisher)](https://itrevolution.com/)
- [DevOps Handbook](https://www.amazon.com/DevOps-Handbook-World-Class-Reliability-Organizations/dp/1942788002)
- [My previous posts on homelab automation](/tags/automation)
