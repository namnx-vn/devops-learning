# Repository Guidelines

## Project Structure & Module Organization

This repository contains DevOps training material rather than executable application code. The root workbook, `DevOps Fresher Labs.docx`, holds hands-on exercises. Presentation decks live under `DevOps Fresher Slides/`, grouped by subject:

- `Slide Linux/` — Linux, Bash, Ansible, and Terraform
- `Slide AWS/`, `Azure/`, and `GCP/` — cloud-platform modules
- `Docker-K8s/` — Docker, Kubernetes, and Helm
- `CICD/` — Git, Jenkins, SonarQube, monitoring, and AWS CI/CD

Add new material to the closest existing subject directory. Create a new directory only for a clearly distinct curriculum area.

## Build, Test, and Development Commands

There is no build system or local runtime. Before submitting changes, use lightweight repository checks:

```sh
find . -type f | sort       # review the complete content inventory
find . -type f -empty       # detect accidentally committed empty files
du -sh .                   # check the repository's total binary size
```

Open every changed PDF or DOCX in a compatible viewer and verify that it renders, links work, and no fonts, diagrams, or pages are missing.

## Content Style & Naming Conventions

Keep slides concise, technically accurate, and consistent with the surrounding module. Use standard product capitalization such as Kubernetes, Docker, Jenkins, Terraform, and Azure. Prefer descriptive filenames following the relevant folder's established pattern, for example `Training AWS - Module 12 - Security.pdf` or `Session 11 Kubernetes Security.pdf`. Avoid ambiguous names such as `final.pdf`, and do not add temporary office files such as `~$document.docx`.

## Validation Guidelines

No automated test framework or coverage target is configured. Validation is editorial and visual. Check spelling, command examples, page order, image clarity, and consistency between the lab workbook and slides. For infrastructure examples, confirm commands and configuration snippets in a disposable environment before publishing them. Ensure examples contain no real credentials, account IDs, private endpoints, or customer data.

## Commit & Pull Request Guidelines

Git history is unavailable in this checkout, so use short, imperative commit subjects, such as `Add GKE storage module` or `Fix Terraform lab commands`. Keep each commit focused on one module or correction. Pull requests should summarize the material changed, identify the affected course/module, and describe validation performed. Link the relevant issue when one exists; include screenshots or exported sample pages when layout changed. Call out large binary replacements and explain why they are necessary.
