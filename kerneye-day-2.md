# Build log · Day 2 🔍

**Date:** October 16, 2026
**Project:** Kerneye (built on [Coroot](https://github.com/coroot/coroot))

---

## What I did

Today I inspected the system in detail.

### 1. Removed outbound usage statistics

I noticed the system sends usage statistics outside. Nothing sensitive, but Kerneye has one rule:

> **Nothing leaves the customer's server.**

So I traced the source and removed it completely from the project.

### 2. Removed the Costs page

Coroot includes a page for cloud costs. Kerneye doesn't need it, because everything runs on the customer's own server.

| Removed | Why |
|---|---|
| Usage statistics | No data should leave the customer's server |
| Costs page (cloud costs) | Kerneye is self-hosted, not a cloud-cost tool |

---

## The plan

1. 🧹 **Clean first:** remove everything I don't need
2. 🔧 **Build second:** add new features

### Why self-hosted matters

- ✅ **Lower cost** for small and mid-sized companies
- ✅ **Data stays inside the company**, which matters for data protection laws in every country

---

## Today's discovery 🔎

While reading the code, I realized the project is focused on:

| Area | Status |
|---|---|
| Server performance analysis | ✅ Covered |
| System issues | ✅ Covered |
| Security analysis | ❌ Missing |

The platform has full performance analysis, but no security analysis.

---

## Next step 🛡️

**Adding a security layer to Kerneye**, so performance and security live in one agent.

---

## Question for you

What's the most important thing to monitor for server security?
Share your thoughts: [@kerneyehq](https://instagram.com/kerneyehq)

---

