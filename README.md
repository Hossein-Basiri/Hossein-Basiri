<h1 align="center">Hossein Basiri</h1>

<p align="center">
  <b>Senior Backend &amp; Platform Engineer</b><br>
  .NET · Azure · Kubernetes · Distributed Systems
</p>

<p align="center">
  <img src="https://img.shields.io/badge/📍_Copenhagen,_Denmark-1f2937?style=flat-square" alt="Copenhagen, Denmark">
  <img src="https://img.shields.io/badge/10%2B_years_shipping_SaaS-1f2937?style=flat-square" alt="10+ years">
  <img src="https://img.shields.io/badge/Backend_Lead-1f2937?style=flat-square" alt="Backend Lead">
</p>

<p align="center">
  <a href="https://www.linkedin.com/in/basiri-hossein"><img src="https://img.shields.io/badge/LinkedIn-0A66C2?style=for-the-badge&logo=linkedin&logoColor=white" alt="LinkedIn"></a>
  <a href="mailto:basiri.hossein@proton.me"><img src="https://img.shields.io/badge/Email-6D4AFF?style=for-the-badge&logo=protonmail&logoColor=white" alt="Email"></a>
  <a href="https://github.com/Hossein-Basiri/semx-showcase"><img src="https://img.shields.io/badge/Featured_project-181717?style=for-the-badge&logo=github&logoColor=white" alt="Featured project"></a>
</p>

---

## About me

Backend Lead / Senior Software Engineer at **Leanlinking**, where I own the backend architecture of a supplier-collaboration SaaS platform used daily by ~20 enterprise customers — including **Roche**, **AMS** and **Bulten**.

I work at the intersection of **distributed systems, event-driven architecture and production reliability**: the person the team turns to for architecture and platform decisions. I also lead the backend for **DEALS**, an AI-powered enterprise negotiation platform built together with Roche across three engineering teams.

Outside of work I build **SEMX** (below), teach **Python &amp; Data Analytics** as a volunteer instructor at **ReDI School**, and use AI-assisted engineering every day — Claude Code, Codex, AI code review, agents and local LLMs.

---

## Featured project — SEMX

> ### 🧾 SEMX — AI-Powered Smart Expense Manager
>
> A production-style personal-finance platform: import a bank statement, get every transaction auto-categorized, track budgets, and receive spend forecasts and anomaly alerts. Built as a **.NET 10 microservices system** behind a single HTTPS gateway, with a **React 19 + TypeScript** front end, PostgreSQL, Docker Compose and end-to-end observability.
>
> **What's interesting about it**
>
> - **Security by design** — RS256 JWTs signed only by the auth service, single-use rotating refresh tokens (replay revokes the whole session family), scope-limited service-to-service tokens, TOTP 2FA, rate-limited auth endpoints.
> - **Forecasting with no ML runtime** — managed Holt-Winters smoothing, split-conformal prediction intervals and a median/MAD anomaly detector, all in pure portable C# (identical on x64 and ARM64), with per-series model selection by backtest.
> - **Layered categorization** — user history → keyword rules → LLM (local Ollama, cloud fallback) only when the cheap tiers can't decide, so most rows never touch a model.
> - **Multi-format import** — CSV, OFX/QFX and PDF statements, or scan a QR code to send one straight from your phone.
> - **Observability & CI** — OpenTelemetry traces, metrics and logs over OTLP; one distributed trace across the gateway and every service; xUnit suites plus a full-stack Docker smoke test in GitHub Actions.
>
> ```
>                    ┌──────────────────────────┐
>                    │   API Gateway (YARP)     │
>                    │ single HTTPS entry point │
>                    └────────────┬─────────────┘
>            /users/**         /expenses/**        /insights/**
>                 ▼                  ▼                  ▼
>          UserService        ExpenseService      InsightService
>            (auth)          (data + import)      (forecasting)
>                 │                  │                  │
>                 ▼                  ▼                  │
>           PostgreSQL         PostgreSQL ◄─────────────┘
>                                            reads aggregates
>                                            & budgets over HTTP
> ```
>
> <p>
>   <img src="https://img.shields.io/badge/.NET_10-512BD4?style=flat-square&logo=dotnet&logoColor=white" alt=".NET 10">
>   <img src="https://img.shields.io/badge/YARP-512BD4?style=flat-square" alt="YARP">
>   <img src="https://img.shields.io/badge/React_19-20232A?style=flat-square&logo=react&logoColor=61DAFB" alt="React 19">
>   <img src="https://img.shields.io/badge/PostgreSQL-4169E1?style=flat-square&logo=postgresql&logoColor=white" alt="PostgreSQL">
>   <img src="https://img.shields.io/badge/Docker-2496ED?style=flat-square&logo=docker&logoColor=white" alt="Docker">
>   <img src="https://img.shields.io/badge/OpenTelemetry-425CC7?style=flat-square&logo=opentelemetry&logoColor=white" alt="OpenTelemetry">
> </p>
>
> **→ [Read the full architecture write-up](https://github.com/Hossein-Basiri/semx-showcase)**
>
> The implementation lives in a private repository. Hiring managers: I'm glad to give a guided, read-only walkthrough — just [email me](mailto:basiri.hossein@proton.me).

---

## Tech stack

**Backend &amp; distributed systems**

![C#](https://img.shields.io/badge/C%23-239120?style=flat-square&logo=csharp&logoColor=white)
![.NET](https://img.shields.io/badge/.NET-512BD4?style=flat-square&logo=dotnet&logoColor=white)
![ASP.NET Core](https://img.shields.io/badge/ASP.NET%20Core-512BD4?style=flat-square&logo=dotnet&logoColor=white)
![gRPC](https://img.shields.io/badge/gRPC-244c5a?style=flat-square&logo=grpc&logoColor=white)
![RabbitMQ](https://img.shields.io/badge/RabbitMQ-FF6600?style=flat-square&logo=rabbitmq&logoColor=white)
![Redis](https://img.shields.io/badge/Redis-DC382D?style=flat-square&logo=redis&logoColor=white)
![SQL Server](https://img.shields.io/badge/SQL%20Server-CC2927?style=flat-square&logo=microsoftsqlserver&logoColor=white)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-4169E1?style=flat-square&logo=postgresql&logoColor=white)
![EF Core](https://img.shields.io/badge/EF%20Core-512BD4?style=flat-square&logo=dotnet&logoColor=white)

**Cloud &amp; platform**

![Azure](https://img.shields.io/badge/Azure-0078D4?style=flat-square&logo=microsoftazure&logoColor=white)
![Azure DevOps](https://img.shields.io/badge/Azure%20DevOps-0078D7?style=flat-square&logo=azuredevops&logoColor=white)
![Kubernetes](https://img.shields.io/badge/Kubernetes-326CE5?style=flat-square&logo=kubernetes&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-2496ED?style=flat-square&logo=docker&logoColor=white)
![GitHub Actions](https://img.shields.io/badge/GitHub%20Actions-2088FF?style=flat-square&logo=githubactions&logoColor=white)
![OpenTelemetry](https://img.shields.io/badge/OpenTelemetry-425CC7?style=flat-square&logo=opentelemetry&logoColor=white)
![Grafana](https://img.shields.io/badge/Grafana-F46800?style=flat-square&logo=grafana&logoColor=white)

**Frontend**

![React](https://img.shields.io/badge/React-20232A?style=flat-square&logo=react&logoColor=61DAFB)
![TypeScript](https://img.shields.io/badge/TypeScript-3178C6?style=flat-square&logo=typescript&logoColor=white)
![JavaScript](https://img.shields.io/badge/JavaScript-F7DF1E?style=flat-square&logo=javascript&logoColor=black)
![Blazor](https://img.shields.io/badge/Blazor-512BD4?style=flat-square&logo=blazor&logoColor=white)

**Architecture &amp; practice**

`Microservices` · `Event-Driven Architecture` · `Saga Pattern` · `CQRS` · `Clean Architecture` · `Domain-Driven Design` · `Vertical Slice` · `CI/CD` · `Observability`

---

## Selected highlights

| | |
|---|---|
| **~33% faster CI** | Cut Azure DevOps build &amp; test times through package caching and parallel pipelines. |
| **Resilient by design** | Event-driven systems with RabbitMQ, the Saga pattern and distributed caching — large workloads handled asynchronously, failures recovered correctly. |
| **Team visibility** | Engineering dashboards surfacing blockers, technical debt and delivery risk, so the whole team owns delivery end to end. |
| **AI in the workflow** | Brought AI-assisted development into daily practice, improving implementation speed, review quality and documentation. |
| **Giving back** | Volunteer instructor in Python &amp; Data Analytics at ReDI School, mentoring diverse student groups. |

---

<p align="center">
  <a href="mailto:basiri.hossein@proton.me"><b>basiri.hossein@proton.me</b></a>
  &nbsp;·&nbsp;
  <a href="https://www.linkedin.com/in/basiri-hossein">LinkedIn</a>
  &nbsp;·&nbsp;
  <a href="https://github.com/Hossein-Basiri/semx-showcase">SEMX</a>
</p>
