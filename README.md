# SadServers Troubleshooting Journal

[![SadServers Level](https://img.shields.io/badge/SadServers-Intermediate-2962FF?style=flat-square&labelColor=FFC400&logo=ansible&logoColor=1A237E&logoSize=auto)](https://sadservers.com/u/Isidro+Sunga+Lim)

## Overview

This repository documents my hands-on troubleshooting practice using SadServers.

SadServers is a Linux, DevOps, and SRE troubleshooting platform where users solve real-world server problems in a lab environment. The platform provides scenarios with broken or misconfigured Linux systems, and the goal is to investigate, fix the issue, and validate the result. SadServers describes its platform as real-world Linux and DevOps scenarios for hands-on learning and technical assessment.

I use this repository to document my learning process, not just the final answers.

The goal is to build a systematic troubleshooting mindset that can be applied to Linux administration, DevOps operations, Docker, Kubernetes, web servers, databases, networking, and production-style incidents.

---

## Why I Created This Repository

I created this repository to strengthen my troubleshooting skills through real server scenarios.

I have a SadServers Pro subscription and my own access key, which allows me to connect to lab instances and troubleshoot them from my laptop. This gives me a more realistic workflow because I can practice using my local terminal, SSH, and standard Linux tools instead of only reading theory or watching videos.

My objective is not to memorize commands.

My objective is to learn how to think clearly when something is broken.

---

## My Troubleshooting Method

For every lab, I follow the same process:

```text
1. Identify the symptom
2. Build the dependency path
3. Verify one layer at a time
4. Find the first failure
5. Fix the first failure
6. Validate the result
7. Write down the lesson learned
```

This helps me avoid guessing and random fixes.

Instead of asking:

```text
What command should I run?
```

I train myself to ask:

```text
What is the next thing that must be true?
How can I verify it?
```

---

## Repository Structure

```text
SadServers-Troubleshooting-Journal/
├── README.md
├── Docker/
│   ├── Guide.md
│   ├── Troubleshooting.md
│   ├── Cheatsheet.md
│   └── Labs/
│       ├── Lab-001-Example.md
│       ├── Lab-002-Example.md
│       └── Lab-003-Example.md
├── Linux/
│   ├── Guide.md
│   ├── Troubleshooting.md
│   ├── Cheatsheet.md
│   └── Labs/
├── Networking/
│   ├── Guide.md
│   ├── Troubleshooting.md
│   ├── Cheatsheet.md
│   └── Labs/
├── Web-Servers/
│   ├── Guide.md
│   ├── Troubleshooting.md
│   ├── Cheatsheet.md
│   └── Labs/
├── Databases/
│   ├── Guide.md
│   ├── Troubleshooting.md
│   ├── Cheatsheet.md
│   └── Labs/
└── Kubernetes/
    ├── Guide.md
    ├── Troubleshooting.md
    ├── Cheatsheet.md
    └── Labs/
```

---

## Lab Writeup Format

Each lab will follow a consistent structure:

```text
Scenario
↓
Symptom
↓
Dependency Path
↓
Verification
↓
First Failure
↓
Fix
↓
Validation
↓
Lessons Learned
↓
Knowledge Check
```

This format helps me document both the solution and the troubleshooting thought process.

---

## Example Troubleshooting Flow

```text
Symptom:
The application is not responding.

Dependency Path:
Client request
↓
Network connectivity
↓
Listening port
↓
Service process
↓
Application configuration
↓
Logs
↓
Validation endpoint
```

Instead of restarting services immediately, I verify each layer one at a time.

Example questions:

```text
Is the server reachable?
Is the port listening?
Which process owns the port?
Is the service running?
What do the logs say?
Does the validation command pass?
```

---

## Main Learning Areas

This repository will include notes and labs for:

- Linux troubleshooting
- Docker troubleshooting
- Networking
- SSH
- System services
- Logs
- Web servers such as Nginx and Apache
- Databases
- Kubernetes
- Disk and filesystem issues
- Permissions and ownership
- Process and port troubleshooting
- Incident-style debugging

SadServers includes many Linux and DevOps scenario categories such as Docker, Kubernetes, databases, web servers, DNS, SSH, SSL, systemd, networking, and more.

---

## How I Use SadServers

My workflow:

```text
Start a SadServers lab
↓
Connect to the instance from my laptop using my own access key
↓
Read the scenario carefully
↓
Identify the symptom
↓
Build the dependency path
↓
Verify each layer
↓
Fix the first failed layer
↓
Run the validation check
↓
Document the lesson learned
```

This helps me practice like a real Linux, DevOps, or SRE engineer responding to an incident.

---

## What This Repository Is Not

This repository is not intended to be a copy of SadServers content.

I rewrite notes in my own words and document my own understanding.

The purpose is learning, reflection, and skill development.

I avoid copying full lab text directly. Instead, I summarize the scenario, document my troubleshooting steps, and explain the concepts in a beginner-friendly way.

---

## Personal Learning Rule

For every problem, I remind myself:

```text
Do not guess.
Do not randomly restart.
Do not blindly copy commands.

Understand the symptom.
Follow the path.
Verify one layer at a time.
Fix the first failure.
Validate the result.
```

---

## Goal

My goal is to become more confident in real-world troubleshooting.

By practicing with SadServers and documenting each lab, I am building a repeatable troubleshooting process that applies to:

```text
Linux administration
DevOps operations
SRE incident response
Cloud infrastructure
Container troubleshooting
Production support
```

This repository is part of my journey to become better at solving real infrastructure problems.
