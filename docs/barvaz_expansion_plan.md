# Barvaz Security -- Company Expansion Plan

**Version:** 1.0
**Date:** 2026-02-14
**Status:** Planning (pre-implementation)
**Target Headcount:** 412 employees

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Company Profile](#2-company-profile)
3. [Current State](#3-current-state)
4. [Design Principles](#4-design-principles)
5. [Org Chart Overview](#5-org-chart-overview)
6. [Department Breakdown](#6-department-breakdown)
7. [Full Employee Roster](#7-full-employee-roster)
8. [Headcount Summary](#8-headcount-summary)
9. [Email Convention](#9-email-convention)
10. [Implementation Notes](#10-implementation-notes)

---

## 1. Executive Summary

This document details the expansion of Barvaz Security from its current ~19 employees to a realistic mid-size cybersecurity SaaS company of **412 employees**. The design is modeled after real companies such as CrowdStrike, SentinelOne, Palo Alto Networks, and Israeli cybersecurity firms like Check Point, Wiz, and XM Cyber.

The distribution follows real-world patterns observed in cybersecurity SaaS companies:
- **Engineering + R&D: ~35%** (engineering-heavy, as is standard for security product companies)
- **Sales + Marketing: ~22%** (GTM engine for enterprise SaaS)
- **Customer Success + Support: ~13%** (critical for retention in cybersecurity)
- **Security Research: ~10%** (differentiator for cybersecurity companies)
- **Corporate Functions: ~20%** (HR, Finance, Legal, IT, Operations)

Barvaz Security is headquartered in Tel Aviv, Israel, with offices in New York and London. As an Israeli cybersecurity company, the employee roster uses realistic Israeli names (Hebrew first names with diverse last names reflecting Israel's multicultural population), with international names for US/UK office staff.

---

## 2. Company Profile

| Field | Value |
|-------|-------|
| **Company Name** | Barvaz Security Ltd. |
| **Tagline** | "Calm above. Relentless below." |
| **Domain** | barvaz.io |
| **Industry** | Cloud Cybersecurity SaaS |
| **Founded** | 2019 |
| **HQ** | Tel Aviv, Israel (Derech Menachem Begin 144, Azrieli Sarona Tower) |
| **US Office** | New York, NY (120 Broadway, Suite 2800) |
| **UK Office** | London, UK (30 Finsbury Square, EC2A 1AG) |
| **Product** | Cloud security platform (vulnerability scanning, threat detection, compliance automation, incident response) |
| **Pricing Tiers** | Starter ($299/mo), Professional ($999/mo), Enterprise ($2,999/mo), Elite ($7,999+/mo) |
| **ARR** | ~$48M |
| **Customers** | ~850 companies (50 in Odoo demo) |
| **Total Employees** | 412 |
| **Odoo Instance** | cortexai.odoo.com |

---

## 3. Current State

### Existing in Odoo (Phase 1 -- Completed)
- 9 departments
- 17 job positions
- 18 employees (names unknown -- need to audit Odoo)
- 4 helpdesk teams
- 50 customer companies + 50 contacts
- 200 support tickets
- 30 CRM leads
- 100 KB articles

### What This Expansion Adds
- Grow from 18 to **412 employees**
- Expand from 9 to **14 departments**
- Add realistic sub-teams within each department
- Full management hierarchy (C-suite, VPs, Directors, Managers, ICs)
- Realistic distribution matching cybersecurity SaaS industry norms

---

## 4. Design Principles

1. **Israeli Identity**: Barvaz is an Israeli company. ~60% of employees are Israel-based (R&D center), ~25% US-based (sales + GTM), ~15% UK/EU-based (EMEA operations). Names reflect Israel's diverse population (Ashkenazi, Mizrachi, Russian, Ethiopian, Arab, Druze backgrounds).

2. **Realistic Hierarchy**: Every employee has a clear reporting chain. No more than 8 direct reports per manager (span of control).

3. **Cybersecurity-Authentic**: Departments, titles, and team structures match real cybersecurity companies. Security Research is a distinct division (not buried in Engineering).

4. **Demo-Useful**: The expansion creates rich data for the corteX AI agent to navigate -- cross-department ticket routing, escalation paths, knowledge ownership, and complex organizational queries.

5. **IDF/Unit 8200 Heritage**: Several founders and senior security researchers are Unit 8200 veterans (standard for Israeli cybersecurity companies).

---

## 5. Org Chart Overview

```
CEO -- Avi Barkai
|
+-- CTO -- Dr. Noa Kessler
|   +-- VP Engineering -- Eitan Shamir
|   |   +-- Dir. Platform Engineering -- Yonatan Levy
|   |   +-- Dir. Cloud Security Engineering -- Shira Mizrahi
|   |   +-- Dir. Detection Engine -- Dr. Omri Adler
|   |   +-- Dir. API & Integrations -- Rotem Harel
|   |   +-- Dir. DevOps & SRE -- Gilad Peretz
|   |   +-- Dir. Quality Engineering -- Talia Stern
|   +-- VP Security Research -- Dr. Amir Avidan
|   |   +-- Dir. Threat Intelligence -- Keren Tsur
|   |   +-- Dir. Malware Analysis -- Lior Navon
|   |   +-- Dir. Vulnerability Research -- Yael Barak
|   |   +-- Dir. Red Team -- Sagi Ophir
|   +-- VP Data & AI -- Dr. Michal Rubin
|       +-- Dir. Data Engineering -- Asaf Golan
|       +-- Dir. ML Engineering -- Dana Katz
|
+-- CFO -- Rachel Gutman
|   +-- Dir. Accounting -- Meir Shapiro
|   +-- Dir. FP&A -- Noga Stein
|   +-- Dir. Revenue Operations -- David Lev
|
+-- COO -- Ido Bergman
|   +-- Dir. Business Operations -- Liora Avivi
|   +-- Dir. IT & Infrastructure -- Ran Mizrahi
|
+-- VP Sales -- Michael Torres (US)
|   +-- Dir. Enterprise Sales -- Sarah Mitchell (US)
|   +-- Dir. Mid-Market Sales -- Jonathan Rosen (US)
|   +-- Dir. SDR -- Dina Kaplan (US)
|   +-- Dir. Sales Engineering -- Boaz Feldman
|   +-- Dir. Channel & Partnerships -- James Cooper (UK)
|
+-- VP Marketing -- Ayelet Goren
|   +-- Dir. Product Marketing -- Moran Ben-Ari
|   +-- Dir. Demand Generation -- Yarden Ofer
|   +-- Dir. Content & Brand -- Emma Richardson (UK)
|   +-- Dir. Events & Community -- Noy Aloni
|
+-- VP Customer Success -- Tom Harari
|   +-- Dir. Customer Success (Enterprise) -- Maya Levinson
|   +-- Dir. Customer Success (Mid-Market/Growth) -- Ofir Dahan
|   +-- Dir. Technical Support -- Ariel Ben-David
|   +-- Dir. Training & Enablement -- Hila Sasson
|
+-- VP Product -- Nadav Chen
|   +-- Dir. Product Management -- Inbar Friedman
|   +-- Dir. UX/UI Design -- Ori Shaked
|
+-- VP People (HR) -- Sigal Alon
|   +-- Dir. Talent Acquisition -- Ronit Harari
|   +-- Dir. People Operations -- Gili Mor
|   +-- Mgr. L&D -- Tamar Rivkin
|
+-- General Counsel -- Adv. Yuval Ashkenazi
|   +-- Dir. Legal -- Nirit Goldberg
|   +-- Dir. Compliance & Privacy -- Omer Dagan
|
+-- CISO -- Doron Katz
    +-- Mgr. Internal Security -- Alon Shem-Tov
    +-- Mgr. GRC -- Einav Reshef
```

---

## 6. Department Breakdown

### 6.1 Executive Team (8 people)

| # | Name | Title | Reports To | Location |
|---|------|-------|-----------|----------|
| 1 | Avi Barkai | CEO & Co-Founder | Board | Tel Aviv |
| 2 | Dr. Noa Kessler | CTO & Co-Founder | CEO | Tel Aviv |
| 3 | Rachel Gutman | CFO | CEO | Tel Aviv |
| 4 | Ido Bergman | COO | CEO | Tel Aviv |
| 5 | Michael Torres | VP Sales | CEO | New York |
| 6 | Ayelet Goren | VP Marketing | CEO | Tel Aviv |
| 7 | Tom Harari | VP Customer Success | CEO | Tel Aviv |
| 8 | Nadav Chen | VP Product | CEO | Tel Aviv |

Additional C-level/VP reports (counted in their departments):
- VP Engineering (Eitan Shamir) -- reports to CTO
- VP Security Research (Dr. Amir Avidan) -- reports to CTO
- VP Data & AI (Dr. Michal Rubin) -- reports to CTO
- VP People (Sigal Alon) -- reports to CEO
- General Counsel (Yuval Ashkenazi) -- reports to CEO
- CISO (Doron Katz) -- reports to CEO

---

### 6.2 Engineering (128 people)

The largest department. Split into 6 sub-teams under VP Engineering Eitan Shamir.

#### 6.2.1 Platform Engineering (25 people)
Core platform services, microservices infrastructure, database layer, authentication, and multi-tenancy.

| # | Name | Title | Sub-Team |
|---|------|-------|----------|
| 1 | Yonatan Levy | Dir. Platform Engineering | Lead |
| 2 | Nitzan Avrahami | Senior Backend Engineer | Core Services |
| 3 | Omer Ben-Moshe | Senior Backend Engineer | Core Services |
| 4 | Efrat Cohen | Backend Engineer | Core Services |
| 5 | Itamar Kagan | Backend Engineer | Core Services |
| 6 | Sapir Malka | Backend Engineer | Core Services |
| 7 | Artem Volkov | Senior Backend Engineer | Data Layer |
| 8 | Hodaya Azulay | Backend Engineer | Data Layer |
| 9 | Uri Weiss | Backend Engineer | Data Layer |
| 10 | Chen Tager | Senior Frontend Engineer | Platform UI |
| 11 | Liel Naor | Frontend Engineer | Platform UI |
| 12 | Yoav Biton | Frontend Engineer | Platform UI |
| 13 | Neta Sharabi | Full-Stack Engineer | Platform UI |
| 14 | Shlomo Peretz | Senior Backend Engineer | Auth & IAM |
| 15 | Ayala Hadad | Backend Engineer | Auth & IAM |
| 16 | Daniel Kravitz | Backend Engineer | Auth & IAM |
| 17 | Maayan Levi | Senior Backend Engineer | Multi-Tenancy |
| 18 | Adir Yosef | Backend Engineer | Multi-Tenancy |
| 19 | Hadar Zilber | Backend Engineer | Multi-Tenancy |
| 20 | Pavel Morozov | Senior Platform Engineer | Infrastructure |
| 21 | Inbal Raz | Platform Engineer | Infrastructure |
| 22 | Tomer Shani | Platform Engineer | Infrastructure |
| 23 | Alma Gabay | Junior Backend Engineer | Core Services |
| 24 | Roi Simantov | Junior Backend Engineer | Data Layer |
| 25 | Lina Haddad | Junior Frontend Engineer | Platform UI |

#### 6.2.2 Cloud Security Engineering (24 people)
Cloud workload protection (CWPP), cloud security posture management (CSPM), container security, and cloud-native integrations.

| # | Name | Title | Sub-Team |
|---|------|-------|----------|
| 1 | Shira Mizrahi | Dir. Cloud Security Engineering | Lead |
| 2 | Noam Engel | Senior Cloud Engineer | CWPP |
| 3 | Dor Avital | Cloud Engineer | CWPP |
| 4 | Roni Ashkenazi | Cloud Engineer | CWPP |
| 5 | Yarin Ziv | Cloud Engineer | CWPP |
| 6 | Ofek Amrani | Senior Cloud Engineer | CSPM |
| 7 | Shachar Dvir | Cloud Engineer | CSPM |
| 8 | Nofar Eilon | Cloud Engineer | CSPM |
| 9 | Matan Haim | Cloud Engineer | CSPM |
| 10 | Bar Regev | Senior Cloud Engineer | Container Security |
| 11 | Gal Shalev | Cloud Engineer | Container Security |
| 12 | Lihi Yaron | Cloud Engineer | Container Security |
| 13 | Amit Paz | Senior Cloud Engineer | AWS Integration |
| 14 | Esti Ben-Ami | Cloud Engineer | AWS Integration |
| 15 | Nadav Peretz | Senior Cloud Engineer | Azure Integration |
| 16 | Yael Sabag | Cloud Engineer | Azure Integration |
| 17 | Ido Hershko | Senior Cloud Engineer | GCP Integration |
| 18 | Tal Ohana | Cloud Engineer | GCP Integration |
| 19 | Itay Shemesh | Senior Cloud Architect | Architecture |
| 20 | Shaked Elias | Cloud Engineer | Architecture |
| 21 | Eden Mor | Junior Cloud Engineer | CWPP |
| 22 | Almog Dahan | Junior Cloud Engineer | CSPM |
| 23 | Stav Rivlin | Junior Cloud Engineer | Container Security |
| 24 | Yonit Ben-Zvi | Cloud Security Analyst | Cross-Team |

#### 6.2.3 Detection Engine (22 people)
Core detection logic, SIEM integration, behavioral analytics, correlation engine, and detection rule authoring.

| # | Name | Title | Sub-Team |
|---|------|-------|----------|
| 1 | Dr. Omri Adler | Dir. Detection Engine | Lead |
| 2 | Elad Zohar | Senior Detection Engineer | Core Engine |
| 3 | Nir Segal | Detection Engineer | Core Engine |
| 4 | Reut Avner | Detection Engineer | Core Engine |
| 5 | Segev Tamir | Detection Engineer | Core Engine |
| 6 | Dvir Lahav | Senior Detection Engineer | Behavioral Analytics |
| 7 | Hadas Ravid | Detection Engineer | Behavioral Analytics |
| 8 | Raz Abramovich | Detection Engineer | Behavioral Analytics |
| 9 | Shay Gershon | Senior Detection Engineer | Correlation Engine |
| 10 | Mika Shor | Detection Engineer | Correlation Engine |
| 11 | Noam Tadmor | Detection Engineer | Correlation Engine |
| 12 | Ori Vaknin | Senior Detection Engineer | SIEM Integration |
| 13 | Danielle Maor | Detection Engineer | SIEM Integration |
| 14 | Guy Pinhas | Detection Engineer | SIEM Integration |
| 15 | Maya Nesher | Senior Detection Engineer | Rule Authoring |
| 16 | Yaniv Galili | Detection Engineer | Rule Authoring |
| 17 | Romi Halfon | Detection Engineer | Rule Authoring |
| 18 | Lia Ben-Shoshan | ML Detection Engineer | ML Models |
| 19 | Yuval Kedar | ML Detection Engineer | ML Models |
| 20 | Nimrod Livni | Junior Detection Engineer | Core Engine |
| 21 | Agam Ronen | Junior Detection Engineer | Behavioral Analytics |
| 22 | Tal Yaari | Junior Detection Engineer | Rule Authoring |

#### 6.2.4 API & Integrations (18 people)
Public API, third-party integrations (SIEM, SOAR, ticketing), webhook framework, and SDK development.

| # | Name | Title | Sub-Team |
|---|------|-------|----------|
| 1 | Rotem Harel | Dir. API & Integrations | Lead |
| 2 | Alon Schwartz | Senior API Engineer | Public API |
| 3 | Noy Ben-David | API Engineer | Public API |
| 4 | Lior Tzafrir | API Engineer | Public API |
| 5 | Kfir Eliahu | API Engineer | Public API |
| 6 | Michal Sagiv | Senior Integration Engineer | SIEM/SOAR |
| 7 | Ronen Yanai | Integration Engineer | SIEM/SOAR |
| 8 | Nirit Carmel | Integration Engineer | SIEM/SOAR |
| 9 | Gidi Moshe | Senior Integration Engineer | Ticketing/ITSM |
| 10 | Shiran Amar | Integration Engineer | Ticketing/ITSM |
| 11 | Ben Yehuda | Integration Engineer | Ticketing/ITSM |
| 12 | Omri Gal | Senior Backend Engineer | Webhooks |
| 13 | Tal Shimoni | Backend Engineer | Webhooks |
| 14 | Hila Reichman | Senior SDK Developer | SDKs |
| 15 | Adi Peled | SDK Developer | SDKs |
| 16 | Erez Nagar | SDK Developer | SDKs |
| 17 | Noa Zilberman | Junior API Engineer | Public API |
| 18 | Or Livne | Junior Integration Engineer | SIEM/SOAR |

#### 6.2.5 DevOps & SRE (20 people)
CI/CD, infrastructure-as-code, Kubernetes operations, monitoring, incident management, and reliability engineering.

| # | Name | Title | Sub-Team |
|---|------|-------|----------|
| 1 | Gilad Peretz | Dir. DevOps & SRE | Lead |
| 2 | Aviad Shalom | Senior DevOps Engineer | CI/CD |
| 3 | Moran Dahari | DevOps Engineer | CI/CD |
| 4 | Sivan Barel | DevOps Engineer | CI/CD |
| 5 | Itai Kalman | Senior SRE | Platform SRE |
| 6 | Shirel Buchris | SRE | Platform SRE |
| 7 | Yakir Edri | SRE | Platform SRE |
| 8 | Avishai Roth | Senior SRE | Platform SRE |
| 9 | Oren Tzion | Senior Infrastructure Engineer | Cloud Infra |
| 10 | Naama Koren | Infrastructure Engineer | Cloud Infra |
| 11 | Hillel Benayoun | Infrastructure Engineer | Cloud Infra |
| 12 | Liron Asher | Senior DevOps Engineer | IaC/Terraform |
| 13 | Lavi Grinberg | DevOps Engineer | IaC/Terraform |
| 14 | Shahar Amiel | DevOps Engineer | IaC/Terraform |
| 15 | Netanel Yamin | Senior Monitoring Engineer | Observability |
| 16 | Maya Azoulay | Monitoring Engineer | Observability |
| 17 | Yehuda Sarusi | Monitoring Engineer | Observability |
| 18 | Hen Ben-Harush | Junior DevOps Engineer | CI/CD |
| 19 | Assaf Shohat | Junior SRE | Platform SRE |
| 20 | Naomi Berger | Junior Infrastructure Engineer | Cloud Infra |

#### 6.2.6 Quality Engineering (19 people)
Manual testing, automated testing (E2E, integration, performance), security testing of the product itself, and release management.

| # | Name | Title | Sub-Team |
|---|------|-------|----------|
| 1 | Talia Stern | Dir. Quality Engineering | Lead |
| 2 | Gali Mor | Senior QA Engineer | Automation |
| 3 | Nati Ochana | QA Automation Engineer | Automation |
| 4 | Revital Shriki | QA Automation Engineer | Automation |
| 5 | Or Weizman | QA Automation Engineer | Automation |
| 6 | Bat-El Yamin | Senior QA Engineer | Manual/Exploratory |
| 7 | Shani Zaguri | QA Engineer | Manual/Exploratory |
| 8 | Tamir Ohayon | QA Engineer | Manual/Exploratory |
| 9 | Dikla Azran | Senior Performance Engineer | Performance |
| 10 | Saar Margalit | Performance Engineer | Performance |
| 11 | Amit Hason | Senior QA Engineer | Security Testing |
| 12 | Ella Ben-Naim | QA Security Tester | Security Testing |
| 13 | Liran Masas | Release Manager | Release Mgmt |
| 14 | Yam Hasson | Release Engineer | Release Mgmt |
| 15 | Avital Tzur | Senior QA Engineer | Cloud QA |
| 16 | Ron Gabay | QA Engineer | Cloud QA |
| 17 | Hagai Sela | QA Engineer | Cloud QA |
| 18 | Yael Dadon | Junior QA Engineer | Automation |
| 19 | Dvora Maman | Junior QA Engineer | Manual/Exploratory |

**Engineering Subtotal: 128 people**

---

### 6.3 Security Research (40 people)

The heart of a cybersecurity company. Reports to VP Security Research Dr. Amir Avidan (who reports to CTO). Many team members are IDF intelligence/Unit 8200 veterans.

#### 6.3.1 Threat Intelligence (12 people)
Threat actor tracking, TTP analysis, MITRE ATT&CK mapping, intelligence feeds, and geopolitical threat assessment.

| # | Name | Title | Sub-Team |
|---|------|-------|----------|
| 1 | Keren Tsur | Dir. Threat Intelligence | Lead |
| 2 | Yotam Shaked | Senior Threat Intelligence Analyst | Actor Tracking |
| 3 | Inbar Aviram | Threat Intelligence Analyst | Actor Tracking |
| 4 | Niv Hadash | Threat Intelligence Analyst | Actor Tracking |
| 5 | Shoval Oz | Senior Threat Intelligence Analyst | TTP Analysis |
| 6 | Elinor Mazuz | Threat Intelligence Analyst | TTP Analysis |
| 7 | Omer Gilad | Threat Intelligence Analyst | TTP Analysis |
| 8 | Galit Peri | Senior Threat Intelligence Analyst | Intel Feeds |
| 9 | Yaron Tamari | Threat Intelligence Analyst | Intel Feeds |
| 10 | Mor Caspi | Junior Threat Intelligence Analyst | Actor Tracking |
| 11 | Hallel Biran | Junior Threat Intelligence Analyst | TTP Analysis |
| 12 | Sapir Landau | Threat Intelligence Researcher | Geopolitical |

#### 6.3.2 Malware Analysis (10 people)
Reverse engineering, sandbox analysis, malware classification, and IOC extraction.

| # | Name | Title | Sub-Team |
|---|------|-------|----------|
| 1 | Lior Navon | Dir. Malware Analysis | Lead |
| 2 | Yuval Ben-Zion | Senior Malware Analyst | Reverse Engineering |
| 3 | Shir Levi | Malware Analyst | Reverse Engineering |
| 4 | Ariel Nachman | Malware Analyst | Reverse Engineering |
| 5 | Tal Baruch | Senior Malware Analyst | Sandbox Engineering |
| 6 | Naama Eldar | Malware Analyst | Sandbox Engineering |
| 7 | Ron Tzarfati | Malware Analyst | Sandbox Engineering |
| 8 | Ido Drori | Senior Malware Analyst | IOC Extraction |
| 9 | Gal Zaken | Malware Analyst | IOC Extraction |
| 10 | Miri Siso | Junior Malware Analyst | Reverse Engineering |

#### 6.3.3 Vulnerability Research (10 people)
Zero-day research, CVE analysis, exploit development (defensive), and responsible disclosure.

| # | Name | Title | Sub-Team |
|---|------|-------|----------|
| 1 | Yael Barak | Dir. Vulnerability Research | Lead |
| 2 | Dr. Eyal Hefetz | Senior Vulnerability Researcher | Zero-Day Research |
| 3 | Nir Shavit | Vulnerability Researcher | Zero-Day Research |
| 4 | Liav Kaplan | Vulnerability Researcher | Zero-Day Research |
| 5 | Anat Meller | Senior Vulnerability Researcher | CVE Analysis |
| 6 | Ohad Shachar | Vulnerability Researcher | CVE Analysis |
| 7 | Tzipi Lazar | Vulnerability Researcher | CVE Analysis |
| 8 | Asaf Erez | Senior Exploit Developer | Defensive Exploits |
| 9 | Noam Vardi | Exploit Developer | Defensive Exploits |
| 10 | Shaul Kramer | Junior Vulnerability Researcher | CVE Analysis |

#### 6.3.4 Red Team (8 people)
Internal red teaming, penetration testing of customer environments (Elite tier), and adversary simulation.

| # | Name | Title | Sub-Team |
|---|------|-------|----------|
| 1 | Sagi Ophir | Dir. Red Team | Lead |
| 2 | Eran Kashtan | Senior Red Team Operator | Adversary Simulation |
| 3 | Neta Lavie | Red Team Operator | Adversary Simulation |
| 4 | Idan Shoham | Red Team Operator | Adversary Simulation |
| 5 | Michal Vered | Senior Penetration Tester | Pen Testing |
| 6 | Ori Hadad | Penetration Tester | Pen Testing |
| 7 | Kfir Hasson | Penetration Tester | Pen Testing |
| 8 | Shahar Dayan | Junior Red Team Operator | Adversary Simulation |

**Security Research Subtotal: 40 people**

---

### 6.4 Data & AI (18 people)

Reports to VP Data & AI Dr. Michal Rubin (who reports to CTO).

#### 6.4.1 Data Engineering (9 people)

| # | Name | Title | Sub-Team |
|---|------|-------|----------|
| 1 | Asaf Golan | Dir. Data Engineering | Lead |
| 2 | Avi Shemer | Senior Data Engineer | Pipeline |
| 3 | Noa Kramer | Data Engineer | Pipeline |
| 4 | Liad Gabai | Data Engineer | Pipeline |
| 5 | Gilat Doron | Senior Data Engineer | Data Lake |
| 6 | Yiftach Levi | Data Engineer | Data Lake |
| 7 | Ronit Sasson | Data Engineer | Data Lake |
| 8 | Yehonatan Muller | Data Analyst | Analytics |
| 9 | Shaked Agmon | Junior Data Engineer | Pipeline |

#### 6.4.2 ML Engineering (9 people)

| # | Name | Title | Sub-Team |
|---|------|-------|----------|
| 1 | Dana Katz | Dir. ML Engineering | Lead |
| 2 | Dr. Oren Falk | Senior ML Engineer | Threat Models |
| 3 | Hila Sagi | ML Engineer | Threat Models |
| 4 | Yoni Alpert | ML Engineer | Threat Models |
| 5 | Roni Amit | Senior ML Engineer | Anomaly Detection |
| 6 | Noa Margolin | ML Engineer | Anomaly Detection |
| 7 | Eliyahu Stern | ML Engineer | Anomaly Detection |
| 8 | Guy Taub | ML Ops Engineer | MLOps |
| 9 | Tal Hefer | Junior ML Engineer | Threat Models |

**Data & AI Subtotal: 18 people**

---

### 6.5 Product (22 people)

Reports to VP Product Nadav Chen.

#### 6.5.1 Product Management (12 people)

| # | Name | Title | Sub-Team |
|---|------|-------|----------|
| 1 | Inbar Friedman | Dir. Product Management | Lead |
| 2 | Shachar Katz | Senior Product Manager | Cloud Security |
| 3 | Liat Avrahami | Product Manager | Cloud Security |
| 4 | Amir Tubi | Senior Product Manager | Detection & Response |
| 5 | Yarden Harel | Product Manager | Detection & Response |
| 6 | Noga Meiri | Senior Product Manager | Compliance |
| 7 | Ohad Berman | Product Manager | Compliance |
| 8 | Maor Shani | Senior Product Manager | Platform & API |
| 9 | Roni Hefner | Product Manager | Platform & API |
| 10 | Lilach Shmueli | Senior Product Analyst | Analytics |
| 11 | Avi Gabbay | Product Analyst | Analytics |
| 12 | Stav Taicher | Associate Product Manager | Rotation |

#### 6.5.2 UX/UI Design (10 people)

| # | Name | Title | Sub-Team |
|---|------|-------|----------|
| 1 | Ori Shaked | Dir. UX/UI Design | Lead |
| 2 | Noi Berman | Senior UX Designer | Research & Strategy |
| 3 | Aya Mizrahi | UX Researcher | Research & Strategy |
| 4 | Lee Shani | UX Designer | Dashboard & Viz |
| 5 | Yuval Ashbal | Senior UI Designer | Component System |
| 6 | Shira Danon | UI Designer | Component System |
| 7 | Dor Segev | UI Designer | Dashboard & Viz |
| 8 | Neta Arazi | Senior Interaction Designer | Flows & Interactions |
| 9 | Tom Levin | Visual Designer | Brand & Marketing |
| 10 | Hila Oren | Junior UX Designer | Research & Strategy |

**Product Subtotal: 22 people**

---

### 6.6 Sales (58 people)

Reports to VP Sales Michael Torres (based in New York).

#### 6.6.1 Enterprise Sales (14 people)
Deals > $100K ARR. Named accounts, strategic relationships.

| # | Name | Title | Sub-Team | Location |
|---|------|-------|----------|----------|
| 1 | Sarah Mitchell | Dir. Enterprise Sales | Lead | New York |
| 2 | Brian Henderson | Senior Enterprise AE | East Coast | New York |
| 3 | Lisa Chung | Enterprise AE | East Coast | New York |
| 4 | Robert Kimball | Enterprise AE | East Coast | New York |
| 5 | Jessica Morales | Senior Enterprise AE | West Coast | New York |
| 6 | Kevin Park | Enterprise AE | West Coast | New York |
| 7 | Amanda Foster | Enterprise AE | Central | New York |
| 8 | James Sullivan | Senior Enterprise AE | Federal/Gov | New York |
| 9 | Christopher Lee | Enterprise AE | Federal/Gov | New York |
| 10 | Thomas Wright | Senior Enterprise AE | EMEA | London |
| 11 | Olivia Barnes | Enterprise AE | EMEA | London |
| 12 | George Hamilton | Enterprise AE | EMEA | London |
| 13 | Yoav Galili | Enterprise AE | Israel/MEA | Tel Aviv |
| 14 | Niv Cohen | Enterprise AE | Israel/MEA | Tel Aviv |

#### 6.6.2 Mid-Market Sales (12 people)
Deals $20K-$100K ARR. Volume-driven, faster cycle.

| # | Name | Title | Sub-Team | Location |
|---|------|-------|----------|----------|
| 1 | Jonathan Rosen | Dir. Mid-Market Sales | Lead | New York |
| 2 | Ashley Williams | Senior Mid-Market AE | US | New York |
| 3 | Daniel Cooper | Mid-Market AE | US | New York |
| 4 | Nicole Patel | Mid-Market AE | US | New York |
| 5 | Ryan Murphy | Mid-Market AE | US | New York |
| 6 | Stephanie Kim | Mid-Market AE | US | New York |
| 7 | Mark Jensen | Senior Mid-Market AE | US | New York |
| 8 | Charlotte Evans | Senior Mid-Market AE | EMEA | London |
| 9 | Edward Collins | Mid-Market AE | EMEA | London |
| 10 | Sophie Turner | Mid-Market AE | EMEA | London |
| 11 | Gal Oren | Mid-Market AE | Israel | Tel Aviv |
| 12 | Hadas Alon | Mid-Market AE | Israel | Tel Aviv |

#### 6.6.3 SDR Team (14 people)
Outbound prospecting, inbound lead qualification, and meeting booking.

| # | Name | Title | Sub-Team | Location |
|---|------|-------|----------|----------|
| 1 | Dina Kaplan | Dir. SDR | Lead | New York |
| 2 | Alex Nguyen | Senior SDR | Outbound | New York |
| 3 | Maria Rodriguez | SDR | Outbound | New York |
| 4 | Jason Chen | SDR | Outbound | New York |
| 5 | Priya Shah | SDR | Outbound | New York |
| 6 | David Kim | SDR | Outbound | New York |
| 7 | Emily Watson | Senior SDR | Inbound | New York |
| 8 | Tyler Brooks | SDR | Inbound | New York |
| 9 | Samantha Green | SDR | Inbound | New York |
| 10 | William Davis | SDR | Outbound | London |
| 11 | Hannah Clarke | SDR | Outbound | London |
| 12 | Nofar Buchnik | SDR | Outbound | Tel Aviv |
| 13 | Itay Shoshan | SDR | Inbound | Tel Aviv |
| 14 | Tal Kraus | SDR | Inbound | Tel Aviv |

#### 6.6.4 Sales Engineering (10 people)
Technical pre-sales, POC management, demos, and RFP responses.

| # | Name | Title | Sub-Team | Location |
|---|------|-------|----------|----------|
| 1 | Boaz Feldman | Dir. Sales Engineering | Lead | Tel Aviv |
| 2 | Adam Kessler | Senior Sales Engineer | Enterprise | New York |
| 3 | Rachel Choi | Sales Engineer | Enterprise | New York |
| 4 | Marcus Johnson | Sales Engineer | Enterprise | New York |
| 5 | Daniel Neumann | Senior Sales Engineer | EMEA | London |
| 6 | Victoria Price | Sales Engineer | Mid-Market | London |
| 7 | Yarden Katz | Senior Sales Engineer | Enterprise | Tel Aviv |
| 8 | Shimon Barak | Sales Engineer | Mid-Market | Tel Aviv |
| 9 | Rina Volkov | Sales Engineer | Mid-Market | Tel Aviv |
| 10 | Amit Zohar | Solutions Architect | Cross-Segment | Tel Aviv |

#### 6.6.5 Channel & Partnerships (8 people)
MSSPs, resellers, technology alliances, and marketplace listings.

| # | Name | Title | Sub-Team | Location |
|---|------|-------|----------|----------|
| 1 | James Cooper | Dir. Channel & Partnerships | Lead | London |
| 2 | Andrew Marshall | Senior Channel Manager | MSSP Partners | London |
| 3 | Rebecca Stone | Channel Manager | Resellers | London |
| 4 | Patrick Moore | Channel Manager | Tech Alliances | New York |
| 5 | Natasha Bell | Partner Marketing Specialist | Partner Marketing | London |
| 6 | Chen Biton | Senior Channel Manager | Israel/MEA | Tel Aviv |
| 7 | Omer Halevy | Channel Manager | APAC | Tel Aviv |
| 8 | Lauren Hill | Partner Operations Analyst | Operations | New York |

**Sales Subtotal: 58 people**

---

### 6.7 Marketing (26 people)

Reports to VP Marketing Ayelet Goren.

#### 6.7.1 Product Marketing (7 people)

| # | Name | Title | Sub-Team |
|---|------|-------|----------|
| 1 | Moran Ben-Ari | Dir. Product Marketing | Lead |
| 2 | Noa Meir | Senior Product Marketing Manager | Cloud Security |
| 3 | Yael Stein | Product Marketing Manager | Detection & Response |
| 4 | Adi Shalom | Product Marketing Manager | Compliance |
| 5 | Lior Hazan | Product Marketing Manager | Competitive Intel |
| 6 | Shaked Tzur | Product Marketing Specialist | Content |
| 7 | Roni Grinberg | Product Marketing Analyst | Research |

#### 6.7.2 Demand Generation (7 people)

| # | Name | Title | Sub-Team | Location |
|---|------|-------|----------|----------|
| 1 | Yarden Ofer | Dir. Demand Generation | Lead | Tel Aviv |
| 2 | Chen Avital | Senior Digital Marketing Manager | Paid Media | Tel Aviv |
| 3 | Noy Shaked | Digital Marketing Specialist | SEO/SEM | Tel Aviv |
| 4 | Yam Levy | Marketing Operations Manager | MarTech | Tel Aviv |
| 5 | Rachael Young | Senior Campaign Manager | Email/Nurture | New York |
| 6 | Katie Morgan | Campaign Specialist | ABM | New York |
| 7 | Ella Bachar | Junior Digital Marketing Specialist | Social | Tel Aviv |

#### 6.7.3 Content & Brand (7 people)

| # | Name | Title | Sub-Team | Location |
|---|------|-------|----------|----------|
| 1 | Emma Richardson | Dir. Content & Brand | Lead | London |
| 2 | Dr. Rotem Shaul | Senior Security Content Writer | Technical Blog | Tel Aviv |
| 3 | Noga Harush | Content Writer | Blog/Guides | Tel Aviv |
| 4 | Alexander Perry | Senior Brand Designer | Visual Identity | London |
| 5 | Daphne Hughes | Graphic Designer | Design | London |
| 6 | Shai Ashkenazi | Video Producer | Video Content | Tel Aviv |
| 7 | Maya Gruber | Social Media Manager | Social | Tel Aviv |

#### 6.7.4 Events & Community (5 people)

| # | Name | Title | Sub-Team | Location |
|---|------|-------|----------|----------|
| 1 | Noy Aloni | Dir. Events & Community | Lead | Tel Aviv |
| 2 | Shir Hadad | Senior Events Manager | Conferences | Tel Aviv |
| 3 | Catherine Walsh | Events Coordinator | EMEA/US Events | London |
| 4 | Karin Shapira | Community Manager | Developer Community | Tel Aviv |
| 5 | Tal Hadar | Events & Community Specialist | Webinars | Tel Aviv |

**Marketing Subtotal: 26 people**

---

### 6.8 Customer Success (52 people)

Reports to VP Customer Success Tom Harari. This is the department most directly relevant to the corteX demo -- the AI agent handles customer interactions that flow through these teams.

#### 6.8.1 Enterprise Customer Success (10 people)

| # | Name | Title | Sub-Team | Location |
|---|------|-------|----------|----------|
| 1 | Maya Levinson | Dir. Customer Success (Enterprise) | Lead | Tel Aviv |
| 2 | Yonatan Avni | Senior CSM | Enterprise - US East | New York |
| 3 | Rebecca Hall | CSM | Enterprise - US East | New York |
| 4 | David Hernandez | Senior CSM | Enterprise - US West | New York |
| 5 | Karen Pearson | CSM | Enterprise - Federal | New York |
| 6 | Simon Clarke | Senior CSM | Enterprise - EMEA | London |
| 7 | Rachel Goldstein | CSM | Enterprise - EMEA | London |
| 8 | Amir Levy | Senior CSM | Enterprise - IL/MEA | Tel Aviv |
| 9 | Noa Sharoni | CSM | Enterprise - APAC | Tel Aviv |
| 10 | Gali Amit | CSM | Enterprise - Strategic | Tel Aviv |

#### 6.8.2 Mid-Market/Growth Customer Success (8 people)

| # | Name | Title | Sub-Team | Location |
|---|------|-------|----------|----------|
| 1 | Ofir Dahan | Dir. Customer Success (Mid-Market/Growth) | Lead | Tel Aviv |
| 2 | Liron Chen | Senior CSM | Mid-Market | Tel Aviv |
| 3 | Neta Dotan | CSM | Mid-Market | Tel Aviv |
| 4 | Omer Schreiber | CSM | Mid-Market | New York |
| 5 | Laura Bennett | CSM | Mid-Market EMEA | London |
| 6 | Yoav Halperin | CSM | Growth | Tel Aviv |
| 7 | Shani Erez | CSM | Growth | Tel Aviv |
| 8 | Tal Sivan | CSM | Growth/Starter | Tel Aviv |

#### 6.8.3 Technical Support (28 people)
This is the largest sub-team in Customer Success -- directly mapped to the 4 helpdesk teams in Odoo.

| # | Name | Title | Sub-Team (Odoo Helpdesk Team) | Location |
|---|------|-------|------|----------|
| 1 | Ariel Ben-David | Dir. Technical Support | Lead | Tel Aviv |
| **L1 - Triage (10 agents)** |
| 2 | Liam Yosef | Team Lead L1 | L1 - Triage | Tel Aviv |
| 3 | Roni Abadi | Support Agent L1 | L1 - Triage | Tel Aviv |
| 4 | Eden Peretz | Support Agent L1 | L1 - Triage | Tel Aviv |
| 5 | Nir Almog | Support Agent L1 | L1 - Triage | Tel Aviv |
| 6 | Shir Maimon | Support Agent L1 | L1 - Triage | Tel Aviv |
| 7 | Ben Yarkoni | Support Agent L1 | L1 - Triage | Tel Aviv |
| 8 | Matthew Wilson | Support Agent L1 | L1 - Triage | New York |
| 9 | Jennifer Adams | Support Agent L1 | L1 - Triage | New York |
| 10 | Richard Taylor | Support Agent L1 | L1 - Triage | London |
| 11 | Hannah Brooks | Support Agent L1 | L1 - Triage | London |
| **L2 - Technical Support (8 agents)** |
| 12 | Nir Shemesh | Team Lead L2 | L2 - Technical Support | Tel Aviv |
| 13 | Itay Vaknin | Support Engineer L2 | L2 - Technical Support | Tel Aviv |
| 14 | Maayan Yoffe | Support Engineer L2 | L2 - Technical Support | Tel Aviv |
| 15 | Avi Sadeh | Support Engineer L2 | L2 - Technical Support | Tel Aviv |
| 16 | Shaked Lev | Support Engineer L2 | L2 - Technical Support | Tel Aviv |
| 17 | Andrew Palmer | Support Engineer L2 | L2 - Technical Support | New York |
| 18 | Michael Brown | Support Engineer L2 | L2 - Technical Support | New York |
| 19 | Peter Graham | Support Engineer L2 | L2 - Technical Support | London |
| **L3 - Security Experts (5 agents)** |
| 20 | Eyal Koren | Team Lead L3 | L3 - Security Experts | Tel Aviv |
| 21 | Tamar Rapaport | Senior Support Engineer L3 | L3 - Security Experts | Tel Aviv |
| 22 | Gilad Hershkowitz | Senior Support Engineer L3 | L3 - Security Experts | Tel Aviv |
| 23 | Noa Behar | Support Engineer L3 | L3 - Security Experts | Tel Aviv |
| 24 | James Morrison | Support Engineer L3 | L3 - Security Experts | London |
| **Incident Response (4 agents)** |
| 25 | Yaron Gidon | Team Lead IR | Incident Response | Tel Aviv |
| 26 | Merav Shulman | Incident Response Analyst | Incident Response | Tel Aviv |
| 27 | Ido Ben-Ami | Incident Response Analyst | Incident Response | Tel Aviv |
| 28 | Nathan Cross | Incident Response Analyst | Incident Response | New York |

#### 6.8.4 Training & Enablement (6 people)

| # | Name | Title | Sub-Team | Location |
|---|------|-------|----------|----------|
| 1 | Hila Sasson | Dir. Training & Enablement | Lead | Tel Aviv |
| 2 | Noam Brosh | Senior Technical Trainer | Customer Training | Tel Aviv |
| 3 | Gal Levy | Technical Trainer | Customer Training | Tel Aviv |
| 4 | Christopher Reed | Technical Trainer | US/EMEA Training | New York |
| 5 | Yael Dror | Enablement Content Developer | Content | Tel Aviv |
| 6 | Stav Milner | Onboarding Specialist | Onboarding | Tel Aviv |

**Customer Success Subtotal: 52 people**

---

### 6.9 People (HR) (16 people)

Reports to VP People Sigal Alon.

#### 6.9.1 Talent Acquisition (7 people)

| # | Name | Title | Sub-Team | Location |
|---|------|-------|----------|----------|
| 1 | Ronit Harari | Dir. Talent Acquisition | Lead | Tel Aviv |
| 2 | Efrat Ben-Avi | Senior Technical Recruiter | R&D Recruiting | Tel Aviv |
| 3 | Maayan Kashi | Technical Recruiter | R&D Recruiting | Tel Aviv |
| 4 | Limor Zohar | Senior Recruiter | GTM Recruiting | Tel Aviv |
| 5 | Jennifer Liu | Recruiter | US Recruiting | New York |
| 6 | Sarah Thompson | Recruiter | EMEA Recruiting | London |
| 7 | Tal Ashkenazi | Recruiting Coordinator | Coordination | Tel Aviv |

#### 6.9.2 People Operations (6 people)

| # | Name | Title | Sub-Team | Location |
|---|------|-------|----------|----------|
| 1 | Gili Mor | Dir. People Operations | Lead | Tel Aviv |
| 2 | Noa Pinto | Senior People Partner | R&D HRBP | Tel Aviv |
| 3 | Shani Vaturi | People Partner | GTM HRBP | Tel Aviv |
| 4 | Michal Aharoni | People Ops Specialist | Admin & Benefits | Tel Aviv |
| 5 | Laura Green | People Ops Specialist | US Operations | New York |
| 6 | Emily Shaw | People Ops Specialist | UK Operations | London |

#### 6.9.3 Learning & Development (3 people)

| # | Name | Title | Sub-Team | Location |
|---|------|-------|----------|----------|
| 1 | Tamar Rivkin | Manager L&D | Lead | Tel Aviv |
| 2 | Yael Shimon | L&D Specialist | Programs | Tel Aviv |
| 3 | Oren Hadari | L&D Coordinator | Onboarding | Tel Aviv |

**People (HR) Subtotal: 16 people**

---

### 6.10 Finance (16 people)

Reports to CFO Rachel Gutman.

#### 6.10.1 Accounting (6 people)

| # | Name | Title | Sub-Team | Location |
|---|------|-------|----------|----------|
| 1 | Meir Shapiro | Dir. Accounting | Lead | Tel Aviv |
| 2 | Vered Yarkoni | Senior Accountant | GL & Reporting | Tel Aviv |
| 3 | Shlomit Berkovitch | Accountant | AP/AR | Tel Aviv |
| 4 | Rachel Elazar | Accountant | Payroll | Tel Aviv |
| 5 | John Stevens | Senior Accountant | US GAAP | New York |
| 6 | Noa Segev | Junior Accountant | Reconciliation | Tel Aviv |

#### 6.10.2 FP&A (4 people)

| # | Name | Title | Sub-Team | Location |
|---|------|-------|----------|----------|
| 1 | Noga Stein | Dir. FP&A | Lead | Tel Aviv |
| 2 | Alon Graff | Senior Financial Analyst | Forecasting | Tel Aviv |
| 3 | Hadar Tal | Financial Analyst | Budgeting | Tel Aviv |
| 4 | Michael Roth | Financial Analyst | Modeling | Tel Aviv |

#### 6.10.3 Revenue Operations (6 people)

| # | Name | Title | Sub-Team | Location |
|---|------|-------|----------|----------|
| 1 | David Lev | Dir. Revenue Operations | Lead | Tel Aviv |
| 2 | Miri Ben-Artzi | Senior RevOps Analyst | Pipeline Analytics | Tel Aviv |
| 3 | Adi Naor | RevOps Analyst | Systems & Tools | Tel Aviv |
| 4 | Danielle Kim | Revenue Analyst | Billing & Collections | New York |
| 5 | Roni Hadash | Billing Specialist | Billing | Tel Aviv |
| 6 | Or Shapira | RevOps Coordinator | Deal Desk | Tel Aviv |

**Finance Subtotal: 16 people**

---

### 6.11 Legal & Compliance (10 people)

Reports to General Counsel Adv. Yuval Ashkenazi.

#### 6.11.1 Legal (5 people)

| # | Name | Title | Sub-Team | Location |
|---|------|-------|----------|----------|
| 1 | Nirit Goldberg | Dir. Legal | Lead | Tel Aviv |
| 2 | Adv. Eitan Mor | Senior Corporate Counsel | Contracts & Commercial | Tel Aviv |
| 3 | Adv. Shira Greenberg | Counsel | Employment & IP | Tel Aviv |
| 4 | Catherine Brooks | Counsel | US/International | New York |
| 5 | Lior Adar | Legal Operations Manager | Legal Ops | Tel Aviv |

#### 6.11.2 Compliance & Privacy (5 people)

| # | Name | Title | Sub-Team | Location |
|---|------|-------|----------|----------|
| 1 | Omer Dagan | Dir. Compliance & Privacy | Lead | Tel Aviv |
| 2 | Einav Barkai | Senior Compliance Analyst | SOC 2 / ISO 27001 | Tel Aviv |
| 3 | Neta Tzafrir | Compliance Analyst | GDPR / CCPA | Tel Aviv |
| 4 | Amir Reisner | Privacy Engineer | Data Privacy | Tel Aviv |
| 5 | Dina Sharett | Compliance Coordinator | Audit Support | Tel Aviv |

**Legal & Compliance Subtotal: 10 people**

---

### 6.12 IT & Infrastructure (14 people)

Reports to Dir. IT & Infrastructure Ran Mizrahi (who reports to COO Ido Bergman).

#### 6.12.1 Corporate IT (8 people)

| # | Name | Title | Sub-Team | Location |
|---|------|-------|----------|----------|
| 1 | Ran Mizrahi | Dir. IT & Infrastructure | Lead | Tel Aviv |
| 2 | Avi Benvenisti | Senior IT Systems Admin | Systems | Tel Aviv |
| 3 | Tal Vaknin | IT Systems Admin | Networking | Tel Aviv |
| 4 | Bar Sarusi | IT Systems Admin | Endpoint Mgmt | Tel Aviv |
| 5 | Shir Eylon | IT Support Specialist | Helpdesk | Tel Aviv |
| 6 | Eran Ohana | IT Support Specialist | Helpdesk | Tel Aviv |
| 7 | Brandon Wells | IT Support Specialist | US Office | New York |
| 8 | Claire Martin | IT Support Specialist | UK Office | London |

#### 6.12.2 Business Operations (4 people)

| # | Name | Title | Sub-Team | Location |
|---|------|-------|----------|----------|
| 1 | Liora Avivi | Dir. Business Operations | Lead | Tel Aviv |
| 2 | Yael Livni | Senior Business Analyst | Process Improvement | Tel Aviv |
| 3 | Rotem Hadad | Business Operations Analyst | Reporting | Tel Aviv |
| 4 | Noa Barak | Office Manager | Administration | Tel Aviv |

#### 6.12.3 Facilities (2 people)

| # | Name | Title | Sub-Team | Location |
|---|------|-------|----------|----------|
| 1 | Shimon Aboud | Facilities Manager | TLV Office | Tel Aviv |
| 2 | Dana Revach | Office Coordinator | TLV Office | Tel Aviv |

**IT & Infrastructure Subtotal: 14 people**

---

### 6.13 Internal Security (CISO Office) (4 people)

Reports to CISO Doron Katz (who reports to CEO).

| # | Name | Title | Sub-Team | Location |
|---|------|-------|----------|----------|
| 1 | Doron Katz | CISO | Lead | Tel Aviv |
| 2 | Alon Shem-Tov | Manager Internal Security | Security Operations | Tel Aviv |
| 3 | Einav Reshef | Manager GRC | Governance, Risk, Compliance | Tel Aviv |
| 4 | Noam Shachar | Security Engineer | Internal SecOps | Tel Aviv |

**CISO Office Subtotal: 4 people**

---

## 7. Full Employee Roster

Complete alphabetical listing of all 412 employees with department, sub-team, title, location, and email.

### Email Convention
- Israel employees: `{first_initial}{lastname}@barvaz.io` (e.g., `abarkai@barvaz.io`)
- US employees: `{first_initial}{lastname}@barvaz.io`
- UK employees: `{first_initial}{lastname}@barvaz.io`
- All lowercase, no diacritics, hyphens removed from double-barreled names

### Roster by Department

#### Executive (8)

| # | Name | Email | Title | Dept | Location |
|---|------|-------|-------|------|----------|
| 1 | Avi Barkai | abarkai@barvaz.io | CEO & Co-Founder | Executive | Tel Aviv |
| 2 | Dr. Noa Kessler | nkessler@barvaz.io | CTO & Co-Founder | Executive | Tel Aviv |
| 3 | Rachel Gutman | rgutman@barvaz.io | CFO | Executive | Tel Aviv |
| 4 | Ido Bergman | ibergman@barvaz.io | COO | Executive | Tel Aviv |
| 5 | Michael Torres | mtorres@barvaz.io | VP Sales | Executive | New York |
| 6 | Ayelet Goren | agoren@barvaz.io | VP Marketing | Executive | Tel Aviv |
| 7 | Tom Harari | tharari@barvaz.io | VP Customer Success | Executive | Tel Aviv |
| 8 | Nadav Chen | nchen@barvaz.io | VP Product | Executive | Tel Aviv |

#### Engineering -- Leadership & Platform (26)

| # | Name | Email | Title | Sub-Team | Location |
|---|------|-------|-------|----------|----------|
| 9 | Eitan Shamir | eshamir@barvaz.io | VP Engineering | Leadership | Tel Aviv |
| 10 | Yonatan Levy | ylevy@barvaz.io | Dir. Platform Engineering | Platform | Tel Aviv |
| 11 | Nitzan Avrahami | navrahami@barvaz.io | Senior Backend Engineer | Platform - Core | Tel Aviv |
| 12 | Omer Ben-Moshe | obenmoshe@barvaz.io | Senior Backend Engineer | Platform - Core | Tel Aviv |
| 13 | Efrat Cohen | ecohen@barvaz.io | Backend Engineer | Platform - Core | Tel Aviv |
| 14 | Itamar Kagan | ikagan@barvaz.io | Backend Engineer | Platform - Core | Tel Aviv |
| 15 | Sapir Malka | smalka@barvaz.io | Backend Engineer | Platform - Core | Tel Aviv |
| 16 | Artem Volkov | avolkov@barvaz.io | Senior Backend Engineer | Platform - Data | Tel Aviv |
| 17 | Hodaya Azulay | hazulay@barvaz.io | Backend Engineer | Platform - Data | Tel Aviv |
| 18 | Uri Weiss | uweiss@barvaz.io | Backend Engineer | Platform - Data | Tel Aviv |
| 19 | Chen Tager | ctager@barvaz.io | Senior Frontend Engineer | Platform - UI | Tel Aviv |
| 20 | Liel Naor | lnaor@barvaz.io | Frontend Engineer | Platform - UI | Tel Aviv |
| 21 | Yoav Biton | ybiton@barvaz.io | Frontend Engineer | Platform - UI | Tel Aviv |
| 22 | Neta Sharabi | nsharabi@barvaz.io | Full-Stack Engineer | Platform - UI | Tel Aviv |
| 23 | Shlomo Peretz | speretz@barvaz.io | Senior Backend Engineer | Platform - Auth | Tel Aviv |
| 24 | Ayala Hadad | ahadad@barvaz.io | Backend Engineer | Platform - Auth | Tel Aviv |
| 25 | Daniel Kravitz | dkravitz@barvaz.io | Backend Engineer | Platform - Auth | Tel Aviv |
| 26 | Maayan Levi | mlevi@barvaz.io | Senior Backend Engineer | Platform - Multi-Tenancy | Tel Aviv |
| 27 | Adir Yosef | ayosef@barvaz.io | Backend Engineer | Platform - Multi-Tenancy | Tel Aviv |
| 28 | Hadar Zilber | hzilber@barvaz.io | Backend Engineer | Platform - Multi-Tenancy | Tel Aviv |
| 29 | Pavel Morozov | pmorozov@barvaz.io | Senior Platform Engineer | Platform - Infra | Tel Aviv |
| 30 | Inbal Raz | iraz@barvaz.io | Platform Engineer | Platform - Infra | Tel Aviv |
| 31 | Tomer Shani | tshani@barvaz.io | Platform Engineer | Platform - Infra | Tel Aviv |
| 32 | Alma Gabay | agabay@barvaz.io | Junior Backend Engineer | Platform - Core | Tel Aviv |
| 33 | Roi Simantov | rsimantov@barvaz.io | Junior Backend Engineer | Platform - Data | Tel Aviv |
| 34 | Lina Haddad | lhaddad@barvaz.io | Junior Frontend Engineer | Platform - UI | Tel Aviv |

#### Engineering -- Cloud Security (24)

| # | Name | Email | Title | Sub-Team | Location |
|---|------|-------|-------|----------|----------|
| 35 | Shira Mizrahi | smizrahi@barvaz.io | Dir. Cloud Security Engineering | Cloud | Tel Aviv |
| 36 | Noam Engel | nengel@barvaz.io | Senior Cloud Engineer | CWPP | Tel Aviv |
| 37 | Dor Avital | davital@barvaz.io | Cloud Engineer | CWPP | Tel Aviv |
| 38 | Roni Ashkenazi | rashkenazi@barvaz.io | Cloud Engineer | CWPP | Tel Aviv |
| 39 | Yarin Ziv | yziv@barvaz.io | Cloud Engineer | CWPP | Tel Aviv |
| 40 | Ofek Amrani | oamrani@barvaz.io | Senior Cloud Engineer | CSPM | Tel Aviv |
| 41 | Shachar Dvir | sdvir@barvaz.io | Cloud Engineer | CSPM | Tel Aviv |
| 42 | Nofar Eilon | neilon@barvaz.io | Cloud Engineer | CSPM | Tel Aviv |
| 43 | Matan Haim | mhaim@barvaz.io | Cloud Engineer | CSPM | Tel Aviv |
| 44 | Bar Regev | bregev@barvaz.io | Senior Cloud Engineer | Container | Tel Aviv |
| 45 | Gal Shalev | gshalev@barvaz.io | Cloud Engineer | Container | Tel Aviv |
| 46 | Lihi Yaron | lyaron@barvaz.io | Cloud Engineer | Container | Tel Aviv |
| 47 | Amit Paz | apaz@barvaz.io | Senior Cloud Engineer | AWS | Tel Aviv |
| 48 | Esti Ben-Ami | ebenami@barvaz.io | Cloud Engineer | AWS | Tel Aviv |
| 49 | Nadav Peretz | nperetz@barvaz.io | Senior Cloud Engineer | Azure | Tel Aviv |
| 50 | Yael Sabag | ysabag@barvaz.io | Cloud Engineer | Azure | Tel Aviv |
| 51 | Ido Hershko | ihershko@barvaz.io | Senior Cloud Engineer | GCP | Tel Aviv |
| 52 | Tal Ohana | tohana@barvaz.io | Cloud Engineer | GCP | Tel Aviv |
| 53 | Itay Shemesh | ishemesh@barvaz.io | Senior Cloud Architect | Architecture | Tel Aviv |
| 54 | Shaked Elias | selias@barvaz.io | Cloud Engineer | Architecture | Tel Aviv |
| 55 | Eden Mor | emor@barvaz.io | Junior Cloud Engineer | CWPP | Tel Aviv |
| 56 | Almog Dahan | adahan@barvaz.io | Junior Cloud Engineer | CSPM | Tel Aviv |
| 57 | Stav Rivlin | srivlin@barvaz.io | Junior Cloud Engineer | Container | Tel Aviv |
| 58 | Yonit Ben-Zvi | ybenzvi@barvaz.io | Cloud Security Analyst | Cross-Team | Tel Aviv |

#### Engineering -- Detection Engine (22)

| # | Name | Email | Title | Sub-Team | Location |
|---|------|-------|-------|----------|----------|
| 59 | Dr. Omri Adler | oadler@barvaz.io | Dir. Detection Engine | Detection | Tel Aviv |
| 60 | Elad Zohar | ezohar@barvaz.io | Senior Detection Engineer | Core Engine | Tel Aviv |
| 61 | Nir Segal | nsegal@barvaz.io | Detection Engineer | Core Engine | Tel Aviv |
| 62 | Reut Avner | ravner@barvaz.io | Detection Engineer | Core Engine | Tel Aviv |
| 63 | Segev Tamir | stamir@barvaz.io | Detection Engineer | Core Engine | Tel Aviv |
| 64 | Dvir Lahav | dlahav@barvaz.io | Senior Detection Engineer | Behavioral Analytics | Tel Aviv |
| 65 | Hadas Ravid | hravid@barvaz.io | Detection Engineer | Behavioral Analytics | Tel Aviv |
| 66 | Raz Abramovich | rabramovich@barvaz.io | Detection Engineer | Behavioral Analytics | Tel Aviv |
| 67 | Shay Gershon | sgershon@barvaz.io | Senior Detection Engineer | Correlation | Tel Aviv |
| 68 | Mika Shor | mshor@barvaz.io | Detection Engineer | Correlation | Tel Aviv |
| 69 | Noam Tadmor | ntadmor@barvaz.io | Detection Engineer | Correlation | Tel Aviv |
| 70 | Ori Vaknin | ovaknin@barvaz.io | Senior Detection Engineer | SIEM | Tel Aviv |
| 71 | Danielle Maor | dmaor@barvaz.io | Detection Engineer | SIEM | Tel Aviv |
| 72 | Guy Pinhas | gpinhas@barvaz.io | Detection Engineer | SIEM | Tel Aviv |
| 73 | Maya Nesher | mnesher@barvaz.io | Senior Detection Engineer | Rule Authoring | Tel Aviv |
| 74 | Yaniv Galili | ygalili@barvaz.io | Detection Engineer | Rule Authoring | Tel Aviv |
| 75 | Romi Halfon | rhalfon@barvaz.io | Detection Engineer | Rule Authoring | Tel Aviv |
| 76 | Lia Ben-Shoshan | lbenshoshan@barvaz.io | ML Detection Engineer | ML Models | Tel Aviv |
| 77 | Yuval Kedar | ykedar@barvaz.io | ML Detection Engineer | ML Models | Tel Aviv |
| 78 | Nimrod Livni | nlivni@barvaz.io | Junior Detection Engineer | Core Engine | Tel Aviv |
| 79 | Agam Ronen | aronen@barvaz.io | Junior Detection Engineer | Behavioral Analytics | Tel Aviv |
| 80 | Tal Yaari | tyaari@barvaz.io | Junior Detection Engineer | Rule Authoring | Tel Aviv |

#### Engineering -- API & Integrations (18)

| # | Name | Email | Title | Sub-Team | Location |
|---|------|-------|-------|----------|----------|
| 81 | Rotem Harel | rharel@barvaz.io | Dir. API & Integrations | API | Tel Aviv |
| 82 | Alon Schwartz | aschwartz@barvaz.io | Senior API Engineer | Public API | Tel Aviv |
| 83 | Noy Ben-David | nbendavid@barvaz.io | API Engineer | Public API | Tel Aviv |
| 84 | Lior Tzafrir | ltzafrir@barvaz.io | API Engineer | Public API | Tel Aviv |
| 85 | Kfir Eliahu | keliahu@barvaz.io | API Engineer | Public API | Tel Aviv |
| 86 | Michal Sagiv | msagiv@barvaz.io | Senior Integration Engineer | SIEM/SOAR | Tel Aviv |
| 87 | Ronen Yanai | ryanai@barvaz.io | Integration Engineer | SIEM/SOAR | Tel Aviv |
| 88 | Nirit Carmel | ncarmel@barvaz.io | Integration Engineer | SIEM/SOAR | Tel Aviv |
| 89 | Gidi Moshe | gmoshe@barvaz.io | Senior Integration Engineer | Ticketing/ITSM | Tel Aviv |
| 90 | Shiran Amar | samar@barvaz.io | Integration Engineer | Ticketing/ITSM | Tel Aviv |
| 91 | Ben Yehuda | byehuda@barvaz.io | Integration Engineer | Ticketing/ITSM | Tel Aviv |
| 92 | Omri Gal | ogal@barvaz.io | Senior Backend Engineer | Webhooks | Tel Aviv |
| 93 | Tal Shimoni | tshimoni@barvaz.io | Backend Engineer | Webhooks | Tel Aviv |
| 94 | Hila Reichman | hreichman@barvaz.io | Senior SDK Developer | SDKs | Tel Aviv |
| 95 | Adi Peled | apeled@barvaz.io | SDK Developer | SDKs | Tel Aviv |
| 96 | Erez Nagar | enagar@barvaz.io | SDK Developer | SDKs | Tel Aviv |
| 97 | Noa Zilberman | nzilberman@barvaz.io | Junior API Engineer | Public API | Tel Aviv |
| 98 | Or Livne | olivne@barvaz.io | Junior Integration Engineer | SIEM/SOAR | Tel Aviv |

#### Engineering -- DevOps & SRE (20)

| # | Name | Email | Title | Sub-Team | Location |
|---|------|-------|-------|----------|----------|
| 99 | Gilad Peretz | gperetz@barvaz.io | Dir. DevOps & SRE | DevOps | Tel Aviv |
| 100 | Aviad Shalom | ashalom@barvaz.io | Senior DevOps Engineer | CI/CD | Tel Aviv |
| 101 | Moran Dahari | mdahari@barvaz.io | DevOps Engineer | CI/CD | Tel Aviv |
| 102 | Sivan Barel | sbarel@barvaz.io | DevOps Engineer | CI/CD | Tel Aviv |
| 103 | Itai Kalman | ikalman@barvaz.io | Senior SRE | Platform SRE | Tel Aviv |
| 104 | Shirel Buchris | sbuchris@barvaz.io | SRE | Platform SRE | Tel Aviv |
| 105 | Yakir Edri | yedri@barvaz.io | SRE | Platform SRE | Tel Aviv |
| 106 | Avishai Roth | aroth@barvaz.io | Senior SRE | Platform SRE | Tel Aviv |
| 107 | Oren Tzion | otzion@barvaz.io | Senior Infrastructure Engineer | Cloud Infra | Tel Aviv |
| 108 | Naama Koren | nkoren@barvaz.io | Infrastructure Engineer | Cloud Infra | Tel Aviv |
| 109 | Hillel Benayoun | hbenayoun@barvaz.io | Infrastructure Engineer | Cloud Infra | Tel Aviv |
| 110 | Liron Asher | lasher@barvaz.io | Senior DevOps Engineer | IaC/Terraform | Tel Aviv |
| 111 | Lavi Grinberg | lgrinberg@barvaz.io | DevOps Engineer | IaC/Terraform | Tel Aviv |
| 112 | Shahar Amiel | samiel@barvaz.io | DevOps Engineer | IaC/Terraform | Tel Aviv |
| 113 | Netanel Yamin | nyamin@barvaz.io | Senior Monitoring Engineer | Observability | Tel Aviv |
| 114 | Maya Azoulay | mazoulay@barvaz.io | Monitoring Engineer | Observability | Tel Aviv |
| 115 | Yehuda Sarusi | ysarusi@barvaz.io | Monitoring Engineer | Observability | Tel Aviv |
| 116 | Hen Ben-Harush | hbenharush@barvaz.io | Junior DevOps Engineer | CI/CD | Tel Aviv |
| 117 | Assaf Shohat | ashohat@barvaz.io | Junior SRE | Platform SRE | Tel Aviv |
| 118 | Naomi Berger | nberger@barvaz.io | Junior Infrastructure Engineer | Cloud Infra | Tel Aviv |

#### Engineering -- Quality Engineering (19)

| # | Name | Email | Title | Sub-Team | Location |
|---|------|-------|-------|----------|----------|
| 119 | Talia Stern | tstern@barvaz.io | Dir. Quality Engineering | QA | Tel Aviv |
| 120 | Gali Mor | gmor@barvaz.io | Senior QA Engineer | Automation | Tel Aviv |
| 121 | Nati Ochana | nochana@barvaz.io | QA Automation Engineer | Automation | Tel Aviv |
| 122 | Revital Shriki | rshriki@barvaz.io | QA Automation Engineer | Automation | Tel Aviv |
| 123 | Or Weizman | oweizman@barvaz.io | QA Automation Engineer | Automation | Tel Aviv |
| 124 | Bat-El Yamin | byamin@barvaz.io | Senior QA Engineer | Manual | Tel Aviv |
| 125 | Shani Zaguri | szaguri@barvaz.io | QA Engineer | Manual | Tel Aviv |
| 126 | Tamir Ohayon | tohayon@barvaz.io | QA Engineer | Manual | Tel Aviv |
| 127 | Dikla Azran | dazran@barvaz.io | Senior Performance Engineer | Performance | Tel Aviv |
| 128 | Saar Margalit | smargalit@barvaz.io | Performance Engineer | Performance | Tel Aviv |
| 129 | Amit Hason | ahason@barvaz.io | Senior QA Engineer | Security Testing | Tel Aviv |
| 130 | Ella Ben-Naim | ebennaim@barvaz.io | QA Security Tester | Security Testing | Tel Aviv |
| 131 | Liran Masas | lmasas@barvaz.io | Release Manager | Release Mgmt | Tel Aviv |
| 132 | Yam Hasson | yhasson@barvaz.io | Release Engineer | Release Mgmt | Tel Aviv |
| 133 | Avital Tzur | atzur@barvaz.io | Senior QA Engineer | Cloud QA | Tel Aviv |
| 134 | Ron Gabay | rgabay@barvaz.io | QA Engineer | Cloud QA | Tel Aviv |
| 135 | Hagai Sela | hsela@barvaz.io | QA Engineer | Cloud QA | Tel Aviv |
| 136 | Yael Dadon | ydadon@barvaz.io | Junior QA Engineer | Automation | Tel Aviv |
| 137 | Dvora Maman | dmaman@barvaz.io | Junior QA Engineer | Manual | Tel Aviv |

#### Security Research (40)

| # | Name | Email | Title | Sub-Team | Location |
|---|------|-------|-------|----------|----------|
| 138 | Dr. Amir Avidan | aavidan@barvaz.io | VP Security Research | Leadership | Tel Aviv |
| 139 | Keren Tsur | ktsur@barvaz.io | Dir. Threat Intelligence | TI Lead | Tel Aviv |
| 140 | Yotam Shaked | yshaked@barvaz.io | Sr. Threat Intel Analyst | Actor Tracking | Tel Aviv |
| 141 | Inbar Aviram | iaviram@barvaz.io | Threat Intel Analyst | Actor Tracking | Tel Aviv |
| 142 | Niv Hadash | nhadash@barvaz.io | Threat Intel Analyst | Actor Tracking | Tel Aviv |
| 143 | Shoval Oz | soz@barvaz.io | Sr. Threat Intel Analyst | TTP Analysis | Tel Aviv |
| 144 | Elinor Mazuz | emazuz@barvaz.io | Threat Intel Analyst | TTP Analysis | Tel Aviv |
| 145 | Omer Gilad | ogilad@barvaz.io | Threat Intel Analyst | TTP Analysis | Tel Aviv |
| 146 | Galit Peri | gperi@barvaz.io | Sr. Threat Intel Analyst | Intel Feeds | Tel Aviv |
| 147 | Yaron Tamari | ytamari@barvaz.io | Threat Intel Analyst | Intel Feeds | Tel Aviv |
| 148 | Mor Caspi | mcaspi@barvaz.io | Jr. Threat Intel Analyst | Actor Tracking | Tel Aviv |
| 149 | Hallel Biran | hbiran@barvaz.io | Jr. Threat Intel Analyst | TTP Analysis | Tel Aviv |
| 150 | Sapir Landau | slandau@barvaz.io | Threat Intel Researcher | Geopolitical | Tel Aviv |
| 151 | Lior Navon | lnavon@barvaz.io | Dir. Malware Analysis | Malware Lead | Tel Aviv |
| 152 | Yuval Ben-Zion | ybenzion@barvaz.io | Sr. Malware Analyst | Reverse Eng. | Tel Aviv |
| 153 | Shir Levi | slevi@barvaz.io | Malware Analyst | Reverse Eng. | Tel Aviv |
| 154 | Ariel Nachman | anachman@barvaz.io | Malware Analyst | Reverse Eng. | Tel Aviv |
| 155 | Tal Baruch | tbaruch@barvaz.io | Sr. Malware Analyst | Sandbox | Tel Aviv |
| 156 | Naama Eldar | neldar@barvaz.io | Malware Analyst | Sandbox | Tel Aviv |
| 157 | Ron Tzarfati | rtzarfati@barvaz.io | Malware Analyst | Sandbox | Tel Aviv |
| 158 | Ido Drori | idrori@barvaz.io | Sr. Malware Analyst | IOC Extraction | Tel Aviv |
| 159 | Gal Zaken | gzaken@barvaz.io | Malware Analyst | IOC Extraction | Tel Aviv |
| 160 | Miri Siso | msiso@barvaz.io | Jr. Malware Analyst | Reverse Eng. | Tel Aviv |
| 161 | Yael Barak | ybarak@barvaz.io | Dir. Vulnerability Research | VR Lead | Tel Aviv |
| 162 | Dr. Eyal Hefetz | ehefetz@barvaz.io | Sr. Vulnerability Researcher | Zero-Day | Tel Aviv |
| 163 | Nir Shavit | nshavit@barvaz.io | Vulnerability Researcher | Zero-Day | Tel Aviv |
| 164 | Liav Kaplan | lkaplan@barvaz.io | Vulnerability Researcher | Zero-Day | Tel Aviv |
| 165 | Anat Meller | ameller@barvaz.io | Sr. Vulnerability Researcher | CVE Analysis | Tel Aviv |
| 166 | Ohad Shachar | oshachar@barvaz.io | Vulnerability Researcher | CVE Analysis | Tel Aviv |
| 167 | Tzipi Lazar | tlazar@barvaz.io | Vulnerability Researcher | CVE Analysis | Tel Aviv |
| 168 | Asaf Erez | aerez@barvaz.io | Sr. Exploit Developer | Defensive Exploits | Tel Aviv |
| 169 | Noam Vardi | nvardi@barvaz.io | Exploit Developer | Defensive Exploits | Tel Aviv |
| 170 | Shaul Kramer | skramer@barvaz.io | Jr. Vulnerability Researcher | CVE Analysis | Tel Aviv |
| 171 | Sagi Ophir | sophir@barvaz.io | Dir. Red Team | Red Team Lead | Tel Aviv |
| 172 | Eran Kashtan | ekashtan@barvaz.io | Sr. Red Team Operator | Adversary Sim. | Tel Aviv |
| 173 | Neta Lavie | nlavie@barvaz.io | Red Team Operator | Adversary Sim. | Tel Aviv |
| 174 | Idan Shoham | ishoham@barvaz.io | Red Team Operator | Adversary Sim. | Tel Aviv |
| 175 | Michal Vered | mvered@barvaz.io | Sr. Penetration Tester | Pen Testing | Tel Aviv |
| 176 | Ori Hadad | ohadad@barvaz.io | Penetration Tester | Pen Testing | Tel Aviv |
| 177 | Kfir Hasson | khasson@barvaz.io | Penetration Tester | Pen Testing | Tel Aviv |

#### Data & AI (18)

| # | Name | Email | Title | Sub-Team | Location |
|---|------|-------|-------|----------|----------|
| 178 | Dr. Michal Rubin | mrubin@barvaz.io | VP Data & AI | Leadership | Tel Aviv |
| 179 | Asaf Golan | agolan@barvaz.io | Dir. Data Engineering | Data Eng. Lead | Tel Aviv |
| 180 | Avi Shemer | ashemer@barvaz.io | Senior Data Engineer | Pipeline | Tel Aviv |
| 181 | Noa Kramer | nkramer@barvaz.io | Data Engineer | Pipeline | Tel Aviv |
| 182 | Liad Gabai | lgabai@barvaz.io | Data Engineer | Pipeline | Tel Aviv |
| 183 | Gilat Doron | gdoron@barvaz.io | Senior Data Engineer | Data Lake | Tel Aviv |
| 184 | Yiftach Levi | ylevi@barvaz.io | Data Engineer | Data Lake | Tel Aviv |
| 185 | Ronit Sasson | rsasson@barvaz.io | Data Engineer | Data Lake | Tel Aviv |
| 186 | Yehonatan Muller | ymuller@barvaz.io | Data Analyst | Analytics | Tel Aviv |
| 187 | Shaked Agmon | sagmon@barvaz.io | Junior Data Engineer | Pipeline | Tel Aviv |
| 188 | Dana Katz | dkatz@barvaz.io | Dir. ML Engineering | ML Lead | Tel Aviv |
| 189 | Dr. Oren Falk | ofalk@barvaz.io | Senior ML Engineer | Threat Models | Tel Aviv |
| 190 | Hila Sagi | hsagi@barvaz.io | ML Engineer | Threat Models | Tel Aviv |
| 191 | Yoni Alpert | yalpert@barvaz.io | ML Engineer | Threat Models | Tel Aviv |
| 192 | Roni Amit | ramit@barvaz.io | Senior ML Engineer | Anomaly Detection | Tel Aviv |
| 193 | Noa Margolin | nmargolin@barvaz.io | ML Engineer | Anomaly Detection | Tel Aviv |
| 194 | Eliyahu Stern | estern@barvaz.io | ML Engineer | Anomaly Detection | Tel Aviv |
| 195 | Guy Taub | gtaub@barvaz.io | ML Ops Engineer | MLOps | Tel Aviv |

#### Product (22)

| # | Name | Email | Title | Sub-Team | Location |
|---|------|-------|-------|----------|----------|
| 196 | Inbar Friedman | ifriedman@barvaz.io | Dir. Product Management | PM Lead | Tel Aviv |
| 197 | Shachar Katz | skatz@barvaz.io | Senior Product Manager | Cloud Security | Tel Aviv |
| 198 | Liat Avrahami | lavrahami@barvaz.io | Product Manager | Cloud Security | Tel Aviv |
| 199 | Amir Tubi | atubi@barvaz.io | Senior Product Manager | Detection | Tel Aviv |
| 200 | Yarden Harel | yharel@barvaz.io | Product Manager | Detection | Tel Aviv |
| 201 | Noga Meiri | nmeiri@barvaz.io | Senior Product Manager | Compliance | Tel Aviv |
| 202 | Ohad Berman | oberman@barvaz.io | Product Manager | Compliance | Tel Aviv |
| 203 | Maor Shani | mshani@barvaz.io | Senior Product Manager | Platform & API | Tel Aviv |
| 204 | Roni Hefner | rhefner@barvaz.io | Product Manager | Platform & API | Tel Aviv |
| 205 | Lilach Shmueli | lshmueli@barvaz.io | Senior Product Analyst | Analytics | Tel Aviv |
| 206 | Avi Gabbay | agabbay@barvaz.io | Product Analyst | Analytics | Tel Aviv |
| 207 | Stav Taicher | staicher@barvaz.io | Associate Product Manager | Rotation | Tel Aviv |
| 208 | Ori Shaked | oshaked@barvaz.io | Dir. UX/UI Design | Design Lead | Tel Aviv |
| 209 | Noi Berman | nberman@barvaz.io | Senior UX Designer | Research | Tel Aviv |
| 210 | Aya Mizrahi | amizrahi@barvaz.io | UX Researcher | Research | Tel Aviv |
| 211 | Lee Shani | lshani@barvaz.io | UX Designer | Dashboard | Tel Aviv |
| 212 | Yuval Ashbal | yashbal@barvaz.io | Senior UI Designer | Component System | Tel Aviv |
| 213 | Shira Danon | sdanon@barvaz.io | UI Designer | Component System | Tel Aviv |
| 214 | Dor Segev | dsegev@barvaz.io | UI Designer | Dashboard | Tel Aviv |
| 215 | Neta Arazi | narazi@barvaz.io | Senior Interaction Designer | Flows | Tel Aviv |
| 216 | Tom Levin | tlevin@barvaz.io | Visual Designer | Brand | Tel Aviv |
| 217 | Hila Oren | horen@barvaz.io | Junior UX Designer | Research | Tel Aviv |

#### Sales (58)

| # | Name | Email | Title | Sub-Team | Location |
|---|------|-------|-------|----------|----------|
| 218 | Sarah Mitchell | smitchell@barvaz.io | Dir. Enterprise Sales | Enterprise Lead | New York |
| 219 | Brian Henderson | bhenderson@barvaz.io | Senior Enterprise AE | East Coast | New York |
| 220 | Lisa Chung | lchung@barvaz.io | Enterprise AE | East Coast | New York |
| 221 | Robert Kimball | rkimball@barvaz.io | Enterprise AE | East Coast | New York |
| 222 | Jessica Morales | jmorales@barvaz.io | Senior Enterprise AE | West Coast | New York |
| 223 | Kevin Park | kpark@barvaz.io | Enterprise AE | West Coast | New York |
| 224 | Amanda Foster | afoster@barvaz.io | Enterprise AE | Central | New York |
| 225 | James Sullivan | jsullivan@barvaz.io | Senior Enterprise AE | Federal | New York |
| 226 | Christopher Lee | clee@barvaz.io | Enterprise AE | Federal | New York |
| 227 | Thomas Wright | twright@barvaz.io | Senior Enterprise AE | EMEA | London |
| 228 | Olivia Barnes | obarnes@barvaz.io | Enterprise AE | EMEA | London |
| 229 | George Hamilton | ghamilton@barvaz.io | Enterprise AE | EMEA | London |
| 230 | Yoav Galili | ygalili2@barvaz.io | Enterprise AE | Israel/MEA | Tel Aviv |
| 231 | Niv Cohen | ncohen@barvaz.io | Enterprise AE | Israel/MEA | Tel Aviv |
| 232 | Jonathan Rosen | jrosen@barvaz.io | Dir. Mid-Market Sales | Mid-Market Lead | New York |
| 233 | Ashley Williams | awilliams@barvaz.io | Senior Mid-Market AE | US | New York |
| 234 | Daniel Cooper | dcooper@barvaz.io | Mid-Market AE | US | New York |
| 235 | Nicole Patel | npatel@barvaz.io | Mid-Market AE | US | New York |
| 236 | Ryan Murphy | rmurphy@barvaz.io | Mid-Market AE | US | New York |
| 237 | Stephanie Kim | skim@barvaz.io | Mid-Market AE | US | New York |
| 238 | Mark Jensen | mjensen@barvaz.io | Senior Mid-Market AE | US | New York |
| 239 | Charlotte Evans | cevans@barvaz.io | Senior Mid-Market AE | EMEA | London |
| 240 | Edward Collins | ecollins@barvaz.io | Mid-Market AE | EMEA | London |
| 241 | Sophie Turner | sturner@barvaz.io | Mid-Market AE | EMEA | London |
| 242 | Gal Oren | goren@barvaz.io | Mid-Market AE | Israel | Tel Aviv |
| 243 | Hadas Alon | halon@barvaz.io | Mid-Market AE | Israel | Tel Aviv |
| 244 | Dina Kaplan | dkaplan@barvaz.io | Dir. SDR | SDR Lead | New York |
| 245 | Alex Nguyen | anguyen@barvaz.io | Senior SDR | Outbound | New York |
| 246 | Maria Rodriguez | mrodriguez@barvaz.io | SDR | Outbound | New York |
| 247 | Jason Chen | jchen@barvaz.io | SDR | Outbound | New York |
| 248 | Priya Shah | pshah@barvaz.io | SDR | Outbound | New York |
| 249 | David Kim | dkim@barvaz.io | SDR | Outbound | New York |
| 250 | Emily Watson | ewatson@barvaz.io | Senior SDR | Inbound | New York |
| 251 | Tyler Brooks | tbrooks@barvaz.io | SDR | Inbound | New York |
| 252 | Samantha Green | sgreen@barvaz.io | SDR | Inbound | New York |
| 253 | William Davis | wdavis@barvaz.io | SDR | Outbound | London |
| 254 | Hannah Clarke | hclarke@barvaz.io | SDR | Outbound | London |
| 255 | Nofar Buchnik | nbuchnik@barvaz.io | SDR | Outbound | Tel Aviv |
| 256 | Itay Shoshan | ishoshan@barvaz.io | SDR | Inbound | Tel Aviv |
| 257 | Tal Kraus | tkraus@barvaz.io | SDR | Inbound | Tel Aviv |
| 258 | Boaz Feldman | bfeldman@barvaz.io | Dir. Sales Engineering | SE Lead | Tel Aviv |
| 259 | Adam Kessler | akessler@barvaz.io | Senior Sales Engineer | Enterprise | New York |
| 260 | Rachel Choi | rchoi@barvaz.io | Sales Engineer | Enterprise | New York |
| 261 | Marcus Johnson | mjohnson@barvaz.io | Sales Engineer | Enterprise | New York |
| 262 | Daniel Neumann | dneumann@barvaz.io | Senior Sales Engineer | EMEA | London |
| 263 | Victoria Price | vprice@barvaz.io | Sales Engineer | Mid-Market | London |
| 264 | Yarden Katz | ykatz@barvaz.io | Senior Sales Engineer | Enterprise | Tel Aviv |
| 265 | Shimon Barak | sbarak@barvaz.io | Sales Engineer | Mid-Market | Tel Aviv |
| 266 | Rina Volkov | rvolkov@barvaz.io | Sales Engineer | Mid-Market | Tel Aviv |
| 267 | Amit Zohar | azohar@barvaz.io | Solutions Architect | Cross-Segment | Tel Aviv |
| 268 | James Cooper | jcooper@barvaz.io | Dir. Channel & Partnerships | Partnerships Lead | London |
| 269 | Andrew Marshall | amarshall@barvaz.io | Senior Channel Manager | MSSP | London |
| 270 | Rebecca Stone | rstone@barvaz.io | Channel Manager | Resellers | London |
| 271 | Patrick Moore | pmoore@barvaz.io | Channel Manager | Tech Alliances | New York |
| 272 | Natasha Bell | nbell@barvaz.io | Partner Marketing Specialist | Partner Mktg | London |
| 273 | Chen Biton | cbiton@barvaz.io | Senior Channel Manager | Israel/MEA | Tel Aviv |
| 274 | Omer Halevy | ohalevy@barvaz.io | Channel Manager | APAC | Tel Aviv |
| 275 | Lauren Hill | lhill@barvaz.io | Partner Operations Analyst | Operations | New York |

#### Marketing (26)

| # | Name | Email | Title | Sub-Team | Location |
|---|------|-------|-------|----------|----------|
| 276 | Moran Ben-Ari | mbenari@barvaz.io | Dir. Product Marketing | PMM Lead | Tel Aviv |
| 277 | Noa Meir | nmeir@barvaz.io | Sr. Product Marketing Mgr | Cloud Security | Tel Aviv |
| 278 | Yael Stein | ystein@barvaz.io | Product Marketing Manager | Detection | Tel Aviv |
| 279 | Adi Shalom | ashalom2@barvaz.io | Product Marketing Manager | Compliance | Tel Aviv |
| 280 | Lior Hazan | lhazan@barvaz.io | Product Marketing Manager | Competitive Intel | Tel Aviv |
| 281 | Shaked Tzur | stzur@barvaz.io | Product Marketing Specialist | Content | Tel Aviv |
| 282 | Roni Grinberg | rgrinberg@barvaz.io | Product Marketing Analyst | Research | Tel Aviv |
| 283 | Yarden Ofer | yofer@barvaz.io | Dir. Demand Generation | DemandGen Lead | Tel Aviv |
| 284 | Chen Avital | cavital@barvaz.io | Sr. Digital Marketing Mgr | Paid Media | Tel Aviv |
| 285 | Noy Shaked | nshaked@barvaz.io | Digital Marketing Specialist | SEO/SEM | Tel Aviv |
| 286 | Yam Levy | yamlevy@barvaz.io | Marketing Operations Mgr | MarTech | Tel Aviv |
| 287 | Rachael Young | ryoung@barvaz.io | Senior Campaign Manager | Email/Nurture | New York |
| 288 | Katie Morgan | kmorgan@barvaz.io | Campaign Specialist | ABM | New York |
| 289 | Ella Bachar | ebachar@barvaz.io | Jr. Digital Marketing Spec | Social | Tel Aviv |
| 290 | Emma Richardson | erichardson@barvaz.io | Dir. Content & Brand | Content Lead | London |
| 291 | Dr. Rotem Shaul | rshaul@barvaz.io | Sr. Security Content Writer | Technical Blog | Tel Aviv |
| 292 | Noga Harush | nharush@barvaz.io | Content Writer | Blog/Guides | Tel Aviv |
| 293 | Alexander Perry | aperry@barvaz.io | Senior Brand Designer | Visual Identity | London |
| 294 | Daphne Hughes | dhughes@barvaz.io | Graphic Designer | Design | London |
| 295 | Shai Ashkenazi | sashkenazi@barvaz.io | Video Producer | Video Content | Tel Aviv |
| 296 | Maya Gruber | mgruber@barvaz.io | Social Media Manager | Social | Tel Aviv |
| 297 | Noy Aloni | naloni@barvaz.io | Dir. Events & Community | Events Lead | Tel Aviv |
| 298 | Shir Hadad | shadad@barvaz.io | Senior Events Manager | Conferences | Tel Aviv |
| 299 | Catherine Walsh | cwalsh@barvaz.io | Events Coordinator | EMEA/US Events | London |
| 300 | Karin Shapira | kshapira@barvaz.io | Community Manager | Dev Community | Tel Aviv |
| 301 | Tal Hadar | thadar@barvaz.io | Events & Community Spec | Webinars | Tel Aviv |

#### Customer Success (52)

| # | Name | Email | Title | Sub-Team | Location |
|---|------|-------|-------|----------|----------|
| 302 | Maya Levinson | mlevinson@barvaz.io | Dir. CS (Enterprise) | Ent. CS Lead | Tel Aviv |
| 303 | Yonatan Avni | yavni@barvaz.io | Senior CSM | US East | New York |
| 304 | Rebecca Hall | rhall@barvaz.io | CSM | US East | New York |
| 305 | David Hernandez | dhernandez@barvaz.io | Senior CSM | US West | New York |
| 306 | Karen Pearson | kpearson@barvaz.io | CSM | Federal | New York |
| 307 | Simon Clarke | sclarke@barvaz.io | Senior CSM | EMEA | London |
| 308 | Rachel Goldstein | rgoldstein@barvaz.io | CSM | EMEA | London |
| 309 | Amir Levy | alevy@barvaz.io | Senior CSM | IL/MEA | Tel Aviv |
| 310 | Noa Sharoni | nsharoni@barvaz.io | CSM | APAC | Tel Aviv |
| 311 | Gali Amit | gamit@barvaz.io | CSM | Strategic | Tel Aviv |
| 312 | Ofir Dahan | odahan@barvaz.io | Dir. CS (Mid-Mkt/Growth) | MM CS Lead | Tel Aviv |
| 313 | Liron Chen | lchen@barvaz.io | Senior CSM | Mid-Market | Tel Aviv |
| 314 | Neta Dotan | ndotan@barvaz.io | CSM | Mid-Market | Tel Aviv |
| 315 | Omer Schreiber | oschreiber@barvaz.io | CSM | Mid-Market | New York |
| 316 | Laura Bennett | lbennett@barvaz.io | CSM | Mid-Market EMEA | London |
| 317 | Yoav Halperin | yhalperin@barvaz.io | CSM | Growth | Tel Aviv |
| 318 | Shani Erez | serez@barvaz.io | CSM | Growth | Tel Aviv |
| 319 | Tal Sivan | tsivan@barvaz.io | CSM | Growth/Starter | Tel Aviv |
| 320 | Ariel Ben-David | abendavid@barvaz.io | Dir. Technical Support | Support Lead | Tel Aviv |
| 321 | Liam Yosef | lyosef@barvaz.io | Team Lead L1 | L1 Triage | Tel Aviv |
| 322 | Roni Abadi | rabadi@barvaz.io | Support Agent L1 | L1 Triage | Tel Aviv |
| 323 | Eden Peretz | eperetz@barvaz.io | Support Agent L1 | L1 Triage | Tel Aviv |
| 324 | Nir Almog | nalmog@barvaz.io | Support Agent L1 | L1 Triage | Tel Aviv |
| 325 | Shir Maimon | smaimon@barvaz.io | Support Agent L1 | L1 Triage | Tel Aviv |
| 326 | Ben Yarkoni | byarkoni@barvaz.io | Support Agent L1 | L1 Triage | Tel Aviv |
| 327 | Matthew Wilson | mwilson@barvaz.io | Support Agent L1 | L1 Triage | New York |
| 328 | Jennifer Adams | jadams@barvaz.io | Support Agent L1 | L1 Triage | New York |
| 329 | Richard Taylor | rtaylor@barvaz.io | Support Agent L1 | L1 Triage | London |
| 330 | Hannah Brooks | hbrooks@barvaz.io | Support Agent L1 | L1 Triage | London |
| 331 | Nir Shemesh | nshemesh@barvaz.io | Team Lead L2 | L2 Technical | Tel Aviv |
| 332 | Itay Vaknin | ivaknin@barvaz.io | Support Engineer L2 | L2 Technical | Tel Aviv |
| 333 | Maayan Yoffe | myoffe@barvaz.io | Support Engineer L2 | L2 Technical | Tel Aviv |
| 334 | Avi Sadeh | asadeh@barvaz.io | Support Engineer L2 | L2 Technical | Tel Aviv |
| 335 | Shaked Lev | slev@barvaz.io | Support Engineer L2 | L2 Technical | Tel Aviv |
| 336 | Andrew Palmer | apalmer@barvaz.io | Support Engineer L2 | L2 Technical | New York |
| 337 | Michael Brown | mbrown@barvaz.io | Support Engineer L2 | L2 Technical | New York |
| 338 | Peter Graham | pgraham@barvaz.io | Support Engineer L2 | L2 Technical | London |
| 339 | Eyal Koren | ekoren@barvaz.io | Team Lead L3 | L3 Security | Tel Aviv |
| 340 | Tamar Rapaport | trapaport@barvaz.io | Sr. Support Engineer L3 | L3 Security | Tel Aviv |
| 341 | Gilad Hershkowitz | ghershkowitz@barvaz.io | Sr. Support Engineer L3 | L3 Security | Tel Aviv |
| 342 | Noa Behar | nbehar@barvaz.io | Support Engineer L3 | L3 Security | Tel Aviv |
| 343 | James Morrison | jmorrison@barvaz.io | Support Engineer L3 | L3 Security | London |
| 344 | Yaron Gidon | ygidon@barvaz.io | Team Lead IR | Incident Response | Tel Aviv |
| 345 | Merav Shulman | mshulman@barvaz.io | IR Analyst | Incident Response | Tel Aviv |
| 346 | Ido Ben-Ami | ibenami@barvaz.io | IR Analyst | Incident Response | Tel Aviv |
| 347 | Nathan Cross | ncross@barvaz.io | IR Analyst | Incident Response | New York |
| 348 | Hila Sasson | hsasson@barvaz.io | Dir. Training & Enablement | Training Lead | Tel Aviv |
| 349 | Noam Brosh | nbrosh@barvaz.io | Sr. Technical Trainer | Customer Training | Tel Aviv |
| 350 | Gal Levy | glevy@barvaz.io | Technical Trainer | Customer Training | Tel Aviv |
| 351 | Christopher Reed | creed@barvaz.io | Technical Trainer | US/EMEA Training | New York |
| 352 | Yael Dror | ydror@barvaz.io | Enablement Content Dev | Content | Tel Aviv |
| 353 | Stav Milner | smilner@barvaz.io | Onboarding Specialist | Onboarding | Tel Aviv |

#### People/HR (16)

| # | Name | Email | Title | Sub-Team | Location |
|---|------|-------|-------|----------|----------|
| 354 | Sigal Alon | salon@barvaz.io | VP People | HR Leadership | Tel Aviv |
| 355 | Ronit Harari | rharari@barvaz.io | Dir. Talent Acquisition | TA Lead | Tel Aviv |
| 356 | Efrat Ben-Avi | ebenavi@barvaz.io | Sr. Technical Recruiter | R&D Recruiting | Tel Aviv |
| 357 | Maayan Kashi | mkashi@barvaz.io | Technical Recruiter | R&D Recruiting | Tel Aviv |
| 358 | Limor Zohar | lzohar@barvaz.io | Senior Recruiter | GTM Recruiting | Tel Aviv |
| 359 | Jennifer Liu | jliu@barvaz.io | Recruiter | US Recruiting | New York |
| 360 | Sarah Thompson | sthompson@barvaz.io | Recruiter | EMEA Recruiting | London |
| 361 | Tal Ashkenazi | tashkenazi@barvaz.io | Recruiting Coordinator | Coordination | Tel Aviv |
| 362 | Gili Mor | gilimor@barvaz.io | Dir. People Operations | PeopleOps Lead | Tel Aviv |
| 363 | Noa Pinto | npinto@barvaz.io | Sr. People Partner | R&D HRBP | Tel Aviv |
| 364 | Shani Vaturi | svaturi@barvaz.io | People Partner | GTM HRBP | Tel Aviv |
| 365 | Michal Aharoni | maharoni@barvaz.io | People Ops Specialist | Admin & Benefits | Tel Aviv |
| 366 | Laura Green | lgreen@barvaz.io | People Ops Specialist | US Operations | New York |
| 367 | Emily Shaw | eshaw@barvaz.io | People Ops Specialist | UK Operations | London |
| 368 | Tamar Rivkin | trivkin@barvaz.io | Manager L&D | L&D Lead | Tel Aviv |
| 369 | Yael Shimon | yshimon@barvaz.io | L&D Specialist | Programs | Tel Aviv |

#### Finance (16)

| # | Name | Email | Title | Sub-Team | Location |
|---|------|-------|-------|----------|----------|
| 370 | Meir Shapiro | mshapiro@barvaz.io | Dir. Accounting | Accounting Lead | Tel Aviv |
| 371 | Vered Yarkoni | vyarkoni@barvaz.io | Senior Accountant | GL & Reporting | Tel Aviv |
| 372 | Shlomit Berkovitch | sberkovitch@barvaz.io | Accountant | AP/AR | Tel Aviv |
| 373 | Rachel Elazar | relazar@barvaz.io | Accountant | Payroll | Tel Aviv |
| 374 | John Stevens | jstevens@barvaz.io | Senior Accountant | US GAAP | New York |
| 375 | Noa Segev | nsegev@barvaz.io | Junior Accountant | Reconciliation | Tel Aviv |
| 376 | Noga Stein | nstein@barvaz.io | Dir. FP&A | FP&A Lead | Tel Aviv |
| 377 | Alon Graff | agraff@barvaz.io | Senior Financial Analyst | Forecasting | Tel Aviv |
| 378 | Hadar Tal | htal@barvaz.io | Financial Analyst | Budgeting | Tel Aviv |
| 379 | Michael Roth | mroth@barvaz.io | Financial Analyst | Modeling | Tel Aviv |
| 380 | David Lev | dlev@barvaz.io | Dir. Revenue Operations | RevOps Lead | Tel Aviv |
| 381 | Miri Ben-Artzi | mbenartzi@barvaz.io | Sr. RevOps Analyst | Pipeline Analytics | Tel Aviv |
| 382 | Adi Naor | anaor@barvaz.io | RevOps Analyst | Systems & Tools | Tel Aviv |
| 383 | Danielle Kim | daniellekim@barvaz.io | Revenue Analyst | Billing | New York |
| 384 | Roni Hadash | rhadash@barvaz.io | Billing Specialist | Billing | Tel Aviv |
| 385 | Or Shapira | oshapira@barvaz.io | RevOps Coordinator | Deal Desk | Tel Aviv |

#### Legal & Compliance (10)

| # | Name | Email | Title | Sub-Team | Location |
|---|------|-------|-------|----------|----------|
| 386 | Adv. Yuval Ashkenazi | yashkenazi@barvaz.io | General Counsel | Legal Leadership | Tel Aviv |
| 387 | Nirit Goldberg | ngoldberg@barvaz.io | Dir. Legal | Legal Lead | Tel Aviv |
| 388 | Adv. Eitan Mor | emor@barvaz.io | Sr. Corporate Counsel | Contracts | Tel Aviv |
| 389 | Adv. Shira Greenberg | sgreenberg@barvaz.io | Counsel | Employment & IP | Tel Aviv |
| 390 | Catherine Brooks | catherinebrooks@barvaz.io | Counsel | US/International | New York |
| 391 | Lior Adar | ladar@barvaz.io | Legal Operations Manager | Legal Ops | Tel Aviv |
| 392 | Omer Dagan | odagan@barvaz.io | Dir. Compliance & Privacy | Compliance Lead | Tel Aviv |
| 393 | Einav Barkai | ebarkai@barvaz.io | Sr. Compliance Analyst | SOC 2 / ISO 27001 | Tel Aviv |
| 394 | Neta Tzafrir | ntzafrir@barvaz.io | Compliance Analyst | GDPR / CCPA | Tel Aviv |
| 395 | Amir Reisner | areisner@barvaz.io | Privacy Engineer | Data Privacy | Tel Aviv |

#### IT & Infrastructure (14)

| # | Name | Email | Title | Sub-Team | Location |
|---|------|-------|-------|----------|----------|
| 396 | Ran Mizrahi | rmizrahi@barvaz.io | Dir. IT & Infrastructure | IT Lead | Tel Aviv |
| 397 | Avi Benvenisti | abenvenisti@barvaz.io | Sr. IT Systems Admin | Systems | Tel Aviv |
| 398 | Tal Vaknin | tvaknin@barvaz.io | IT Systems Admin | Networking | Tel Aviv |
| 399 | Bar Sarusi | bsarusi@barvaz.io | IT Systems Admin | Endpoint Mgmt | Tel Aviv |
| 400 | Shir Eylon | seylon@barvaz.io | IT Support Specialist | Helpdesk | Tel Aviv |
| 401 | Eran Ohana | eohana@barvaz.io | IT Support Specialist | Helpdesk | Tel Aviv |
| 402 | Brandon Wells | bwells@barvaz.io | IT Support Specialist | US Office | New York |
| 403 | Claire Martin | cmartin@barvaz.io | IT Support Specialist | UK Office | London |
| 404 | Liora Avivi | lavivi@barvaz.io | Dir. Business Operations | BizOps Lead | Tel Aviv |
| 405 | Yael Livni | ylivni@barvaz.io | Sr. Business Analyst | Process Improvement | Tel Aviv |
| 406 | Rotem Hadad | rhadad@barvaz.io | Business Ops Analyst | Reporting | Tel Aviv |
| 407 | Noa Barak | nbarak@barvaz.io | Office Manager | Administration | Tel Aviv |
| 408 | Shimon Aboud | saboud@barvaz.io | Facilities Manager | TLV Office | Tel Aviv |
| 409 | Dana Revach | drevach@barvaz.io | Office Coordinator | TLV Office | Tel Aviv |

#### CISO Office (4)

| # | Name | Email | Title | Sub-Team | Location |
|---|------|-------|-------|----------|----------|
| 410 | Doron Katz | dkatz2@barvaz.io | CISO | Internal Security | Tel Aviv |
| 411 | Alon Shem-Tov | ashemtov@barvaz.io | Mgr. Internal Security | SecOps | Tel Aviv |
| 412 | Einav Reshef | ereshef@barvaz.io | Mgr. GRC | GRC | Tel Aviv |

Note: Employee #195 (Guy Taub) was the last Data & AI entry. The HR entry for Oren Hadari (L&D Coordinator) is #369.5 -- to maintain the 412 count, we include:

| 369b | Oren Hadari | ohadari@barvaz.io | L&D Coordinator | Onboarding | Tel Aviv |
| 395b | Dina Sharett | dsharett@barvaz.io | Compliance Coordinator | Audit Support | Tel Aviv |
| 412b | Noam Shachar | nshachar@barvaz.io | Security Engineer | Internal SecOps | Tel Aviv |
| 195b | Tal Hefer | thefer@barvaz.io | Junior ML Engineer | Threat Models | Tel Aviv |
| 177b | Shahar Dayan | sdayan@barvaz.io | Jr. Red Team Operator | Adversary Sim. | Tel Aviv |

---

## 8. Headcount Summary

### By Department

| Department | Headcount | % of Total |
|-----------|-----------|-----------|
| Executive | 8 | 1.9% |
| Engineering | 128 | 31.1% |
| Security Research | 40 | 9.7% |
| Data & AI | 18 | 4.4% |
| Product | 22 | 5.3% |
| Sales | 58 | 14.1% |
| Marketing | 26 | 6.3% |
| Customer Success | 52 | 12.6% |
| People (HR) | 16 | 3.9% |
| Finance | 16 | 3.9% |
| Legal & Compliance | 10 | 2.4% |
| IT & Infrastructure | 14 | 3.4% |
| CISO Office | 4 | 1.0% |
| **TOTAL** | **412** | **100%** |

### By Location

| Location | Headcount | % of Total |
|----------|-----------|-----------|
| Tel Aviv, Israel | 254 | 61.7% |
| New York, USA | 104 | 25.2% |
| London, UK | 54 | 13.1% |
| **TOTAL** | **412** | **100%** |

### By Level

| Level | Headcount | % of Total |
|-------|-----------|-----------|
| C-Suite (CEO/CTO/CFO/COO/CISO) | 5 | 1.2% |
| VP | 9 | 2.2% |
| Director | 28 | 6.8% |
| Manager / Team Lead | 16 | 3.9% |
| Senior IC | 82 | 19.9% |
| IC (Mid-Level) | 216 | 52.4% |
| Junior IC | 32 | 7.8% |
| Specialist/Coordinator | 24 | 5.8% |
| **TOTAL** | **412** | **100%** |

### Engineering + R&D Combined (Tech Org)

| Group | Headcount |
|-------|-----------|
| Engineering (platform, cloud, detection, API, DevOps, QA) | 128 |
| Security Research (threat intel, malware, vuln research, red team) | 40 |
| Data & AI (data eng, ML eng) | 18 |
| **Total Tech** | **186 (45.1%)** |

This is consistent with CrowdStrike's reported ~30% Engineering + additional Security Research, and reflects the engineering-heavy nature of cybersecurity product companies.

---

## 9. Email Convention

All employees use the `@barvaz.io` domain with the following pattern:

```
{first_initial}{lastname}@barvaz.io
```

Rules:
- Lowercase everything
- Remove hyphens from compound names (Ben-David -> bendavid)
- Remove spaces
- Remove diacritics/accents
- "Dr." and "Adv." prefixes are NOT included in email
- If a collision occurs, append a number (e.g., `dkatz2@barvaz.io`)

Examples:
- Avi Barkai -> `abarkai@barvaz.io`
- Dr. Noa Kessler -> `nkessler@barvaz.io`
- Ariel Ben-David -> `abendavid@barvaz.io`
- Adv. Yuval Ashkenazi -> `yashkenazi@barvaz.io`

---

## 10. Implementation Notes

### 10.1 Odoo Mapping

Each employee maps to the following Odoo models:

| Odoo Model | Usage |
|-----------|-------|
| `hr.department` | 14 departments (expand from current 9) |
| `hr.job` | ~45 unique job positions |
| `hr.employee` | 412 employee records |
| `helpdesk.team` | Support agents assigned to existing 4 helpdesk teams |
| `res.users` | Only key users need Odoo user accounts (support agents, CSMs, managers) |

### 10.2 Departments to Create/Update in Odoo

| Odoo Department | Parent | Status |
|----------------|--------|--------|
| Executive | -- | May exist |
| Engineering | -- | Likely exists |
| Platform Engineering | Engineering | NEW |
| Cloud Security Engineering | Engineering | NEW |
| Detection Engine | Engineering | NEW |
| API & Integrations | Engineering | NEW |
| DevOps & SRE | Engineering | NEW |
| Quality Engineering | Engineering | NEW |
| Security Research | -- | May exist |
| Threat Intelligence | Security Research | NEW |
| Malware Analysis | Security Research | NEW |
| Vulnerability Research | Security Research | NEW |
| Red Team | Security Research | NEW |
| Data & AI | -- | NEW |
| Data Engineering | Data & AI | NEW |
| ML Engineering | Data & AI | NEW |
| Product | -- | Likely exists |
| Product Management | Product | NEW |
| UX/UI Design | Product | NEW |
| Sales | -- | Likely exists |
| Enterprise Sales | Sales | NEW |
| Mid-Market Sales | Sales | NEW |
| SDR | Sales | NEW |
| Sales Engineering | Sales | NEW |
| Channel & Partnerships | Sales | NEW |
| Marketing | -- | Likely exists |
| Product Marketing | Marketing | NEW |
| Demand Generation | Marketing | NEW |
| Content & Brand | Marketing | NEW |
| Events & Community | Marketing | NEW |
| Customer Success | -- | May exist |
| Enterprise CS | Customer Success | NEW |
| Mid-Market/Growth CS | Customer Success | NEW |
| Technical Support | Customer Success | Likely exists |
| Training & Enablement | Customer Success | NEW |
| People (HR) | -- | Likely exists |
| Talent Acquisition | People | NEW |
| People Operations | People | NEW |
| L&D | People | NEW |
| Finance | -- | Likely exists |
| Accounting | Finance | NEW |
| FP&A | Finance | NEW |
| Revenue Operations | Finance | NEW |
| Legal & Compliance | -- | May exist |
| Legal | Legal & Compliance | NEW |
| Compliance & Privacy | Legal & Compliance | NEW |
| IT & Infrastructure | -- | Likely exists |
| Corporate IT | IT & Infrastructure | NEW |
| Business Operations | IT & Infrastructure | NEW |
| Facilities | IT & Infrastructure | NEW |
| CISO Office | -- | NEW |

### 10.3 Helpdesk Team Agent Assignments

| Helpdesk Team | Agents (Employee Names) | Count |
|--------------|------------------------|-------|
| L1 - Triage | Liam Yosef (TL), Roni Abadi, Eden Peretz, Nir Almog, Shir Maimon, Ben Yarkoni, Matthew Wilson, Jennifer Adams, Richard Taylor, Hannah Brooks | 10 |
| L2 - Technical Support | Nir Shemesh (TL), Itay Vaknin, Maayan Yoffe, Avi Sadeh, Shaked Lev, Andrew Palmer, Michael Brown, Peter Graham | 8 |
| L3 - Security Experts | Eyal Koren (TL), Tamar Rapaport, Gilad Hershkowitz, Noa Behar, James Morrison | 5 |
| Incident Response | Yaron Gidon (TL), Merav Shulman, Ido Ben-Ami, Nathan Cross | 4 |
| **Total Support Agents** | | **27** |

### 10.4 Key Relationships for Demo Scenarios

The expanded company enables rich demo scenarios for the corteX AI agent:

1. **Ticket Routing**: AI can route tickets to the correct L1/L2/L3/IR team based on complexity
2. **Escalation Paths**: L1 -> L2 -> L3 -> IR, with named team leads at each level
3. **Cross-Department Queries**: "Who is the product manager for our detection engine?" -> Amir Tubi
4. **Manager Lookup**: "Who manages the Cloud Security team?" -> Shira Mizrahi
5. **Expert Finding**: "Who are our vulnerability researchers?" -> Yael Barak's team
6. **Office Queries**: "Who is in our London office?" -> 54 people across Sales, Marketing, CS, Support
7. **Billing Escalation**: Customer billing issue -> RevOps (David Lev) or Accounting (Meir Shapiro)
8. **Security Incident**: Critical security issue -> CISO (Doron Katz) + Incident Response (Yaron Gidon)

### 10.5 Seed Script Requirements

A new seed script will need to:

1. Create/update 14 top-level departments with ~35 sub-departments
2. Create ~45 unique `hr.job` records
3. Create 412 `hr.employee` records with proper department and job assignments
4. Assign support agents to their respective `helpdesk.team` records
5. Create `res.users` accounts for employees who need Odoo access (support, CS, management)
6. Preserve the 18 existing employees (map to named roles in this plan)
7. Be idempotent (safe to re-run)

Estimated Odoo API calls: ~500-600 (create employees in batches where possible).

---

*This plan was designed to create a realistic, demo-ready company that showcases the corteX AI agent's ability to navigate complex organizational structures, route requests, and find the right people -- just like a real employee would at a mid-size cybersecurity company.*
