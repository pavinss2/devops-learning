# Module 0: What is DevOps and Why for Data Engineers

> **Track:** Core Track
> **Duration:** 0.5–1 day
> **Level:** Beginner
> **Prerequisites:** None

---

## Table of Contents

1. [Start Here: A Story You Might Recognize](#1-start-here-a-story-you-might-recognize)
2. [So What is DevOps](#2-so-what-is-devops)
3. [The DevOps Lifecycle](#3-the-devops-lifecycle)
4. [Why Does This Matter for Data Engineers Specifically](#4-why-does-this-matter-for-data-engineers-specifically)
5. [What You Will Learn in This Track](#5-what-you-will-learn-in-this-track)
6. [What DevOps is Not](#6-what-devops-is-not)
7. [How to Approach This Training](#7-how-to-approach-this-training)

---

## 1. Start Here: A Story You Might Recognize

Imagine you just finished building a data pipeline. It took you two weeks. It runs perfectly on your laptop. You are proud of it.

Now you need to get it running on the client's environment.

You send the files to the client's IT team. They set it up on a server. It crashes immediately because they have a different Python version. You fix that. Then it crashes because a library version is different. You fix that. Then it runs but connects to the wrong database because the config file has different settings. You fix that too. Three days later, it is finally running.

Two weeks later, your colleague makes a small change to the pipeline. They copy the new file directly onto the server. It breaks something. Nobody is sure what changed or when. You spend a day debugging.

A month later, the client wants the same pipeline in their staging environment and their production environment. You manually copy everything again. The environments slowly drift apart. Bugs appear in production that do not exist in staging.

Sound familiar? This is the problem DevOps solves.

---

## 2. So What is DevOps

DevOps is not a tool. It is not a job title. It is not something only "DevOps engineers" do.

DevOps is a set of **practices and tools** that help teams deliver software reliably, repeatedly, and quickly. The name comes from combining **Dev** (development; writing code) and **Ops** (operations; running and maintaining that code in production).

Before DevOps existed as a concept, development and operations were two separate teams that often worked against each other:

- Developers wrote code and threw it over the wall to operations
- Operations had to figure out how to run it, often with no documentation
- When something broke in production, each side blamed the other

DevOps is the idea that development and operations should work together, use shared tools, and automate the handoff between writing code and running it in production.

For you as a data engineer, think of it this way:

> **DevOps is the practice of making sure your code gets from your laptop to the client's production environment reliably, automatically, and without surprises.**

---

## 3. The DevOps Lifecycle

DevOps is often described as a continuous loop of eight phases. You do not need to memorize these; just read through them to understand the big picture.

```
      Plan
     /    \
 Operate   Code
    |        |
  Monitor  Build
    |        |
  Deploy   Test
     \    /
      Release
```

| Phase | What happens | Example in data engineering |
|---|---|---|
| **Plan** | Decide what to build; track tasks | Creating a work item in Azure Boards for a new ingestion pipeline |
| **Code** | Write the actual code | Writing a Python ingestion script or a dbt SQL model |
| **Build** | Package the code into something deployable | Building a Docker image of your pipeline |
| **Test** | Automatically verify the code works | Running unit tests and data quality checks |
| **Release** | Prepare the tested code for deployment | Tagging a version; approving a PR |
| **Deploy** | Put the code into the target environment | Running `terraform apply` to provision infrastructure; deploying the pipeline |
| **Operate** | Keep the system running | Managing Kubernetes pods; scaling resources |
| **Monitor** | Watch for problems and learn from them | Alerts when a pipeline fails; dashboards showing run times |

The loop is continuous because software is never "done." You plan, code, deploy, monitor, discover something to improve, plan again.

---

## 4. Why Does This Matter for Data Engineers Specifically

You were hired as a data engineer; to build pipelines, transform data, and make data available for analysis. So why should you care about DevOps?

Three honest reasons:

### Reason 1: Clients expect it

When you join a client as a consultant, you are not just delivering SQL files and Python scripts. You are delivering a working, maintainable, production-grade data system. Clients expect:

- Code that is version-controlled and reviewable
- Deployments that are automated and repeatable
- Infrastructure that is documented and reproducible
- Pipelines that can be monitored and debugged when something goes wrong

If you show up knowing only how to write the code but not how to deliver it properly, you will be dependent on whoever handles the "DevOps side" at the client; and on many engagements, that person does not exist. It falls on you.

### Reason 2: It makes your own work easier

Once you have set up proper CI/CD for a project, you stop doing repetitive manual deployments. You stop asking "which version is running in production?" You stop spending evenings fixing things that broke because someone manually edited a file on a server.

The upfront investment in DevOps practices saves you significant time and stress over the course of a project.

### Reason 3: It protects you professionally

Imagine you make a change to a pipeline, deploy it manually, and something goes wrong in production. Without version control and automated deployments, it is very hard to:

- Prove exactly what changed
- Roll back to the previous version quickly
- Show the client a clear audit trail of what happened

With proper DevOps practices, all of these are straightforward. You have Git history, pipeline logs, and a repeatable rollback process. This protects both you and your client.

---

## 5. What You Will Learn in This Track

This training track covers the DevOps skills most commonly needed on data engineering consultancy projects. Here is a plain-English summary of each module:

**Core Track (the essentials; 2–4 weeks):**

| Module | What it is | Why you need it |
|---|---|---|
| **1: Git** | Version control for your code | Every other module depends on this; all code lives in Git |
| **2: Terraform** | Writing infrastructure as code | So you can provision cloud resources properly instead of clicking through portals |
| **3: CI/CD Fundamentals** | How automated pipelines work | The concepts behind everything in Modules 5A and 5B |
| **4 (Azure) or 4 (non-Azure): Azure Pipelines or GitHub Actions** | The actual CI/CD tool | Automate testing and deployment of your pipelines |
| **6: Secret Management** | Handling passwords and credentials safely | One of the most common mistakes on client projects |
| **7: Docker** | Packaging your code into containers | The standard way to deploy data pipelines in enterprise environments |

**Intermediate Extension (for more complex projects; +2–4 weeks):**

| Module | What it is | Why you need it |
|---|---|---|
| **8: Kubernetes** | Running containers at scale | Enterprise data platforms often run on Kubernetes |
| **9: Advanced dbt CI/CD** | dbt-specific deployment patterns | For projects where dbt is the core transformation tool |
| **10: Monitoring and Alerting** | Watching pipelines in production | So you know about problems before the client does |

---

## 6. What DevOps is Not

A few things worth clarifying upfront so you do not feel intimidated:

**DevOps is not about becoming a DevOps engineer.**
You are still a data engineer. This training gives you enough DevOps knowledge to be self-sufficient on client projects. You are not expected to design cloud network architectures or manage Kubernetes clusters at an admin level.

**DevOps is not just about tools.**
The tools (Git, Terraform, Docker) are the easy part. The harder part is the habit: always using version control, always automating deployments, always thinking about how your code will behave in production. The tools only work if you use them consistently.

**DevOps is not optional on professional engagements.**
This is not extra credit. On serious client projects, these practices are baseline expectations. The goal of this track is to bring everyone up to that baseline.

**You do not need to know everything before starting.**
Every module in this track starts from zero. You will learn by doing; reading concepts, seeing examples, and completing labs. Mistakes are expected and are part of the learning process.

---

## 7. How to Approach This Training

A few practical tips before you dive into Module 1:

**Do the labs.** The concept sections explain what and why. The labs are where the understanding actually sticks. Do not skip them even when they feel simple.

**Use your terminal every day.** Git, Terraform, Docker, kubectl; these are all command-line tools. The more you type these commands, the more natural they become. Discomfort with the terminal fades quickly with practice.

**Break things on purpose.** The labs sometimes ask you to introduce failures; a bad config, a failing test, a conflict. These are the most valuable exercises. Understanding why something breaks teaches you far more than watching it succeed.

**Ask questions early.** If a concept is unclear, ask before moving to the next module. Each module builds on the previous one; a gap in understanding compounds quickly.

**Relate it to your work.** As you go through each module, think about a real project you have worked on. Where would Git have helped? What would a CI/CD pipeline have caught? Connecting concepts to real situations makes them easier to remember.

---

*Ready to begin. Start with Module 1: Version Control with Git.*
