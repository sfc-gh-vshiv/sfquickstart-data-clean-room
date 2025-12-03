# Walmart Data Clean Room: Execution Flow & Cost Guide

A concise guide explaining where queries run and who pays in a Snowflake Data Clean Room between **Walmart** and **Advertisers**.

---

## Quick Summary

| Party | What They Do | Pays For |
|-------|-------------|----------|
| **Advertiser** (Provider) | Hosts ad exposure data, validates requests, approves queries | Tasks, request processing, template validation |
| **Walmart** (Consumer) | Submits requests, executes approved queries | Query execution, data scanning, result generation |

> **Key Insight**: Walmart pays for the heavy lifting (query execution), while the Advertiser pays for lightweight validation tasks.

---

## Architecture Overview

```mermaid
%%{init: {'theme': 'base', 'themeVariables': {'lineColor': '#000000', 'primaryColor': '#e6f7ff'}}}%%
flowchart TB
    subgraph ADVERTISER["📺 ADVERTISER ACCOUNT (Provider)"]
        A1["Advertiser Data"] --- A2["🛡️ Data Firewall"] --- A3["Templates"] --- A4["Request Log"]
    end
    
    ADVERTISER ===>|"1️⃣ Share templates & protected views"| WALMART
    
    subgraph WALMART["🛒 WALMART ACCOUNT (Consumer)"]
        W1["Mounted Share"] --- W2["Walmart Data"] --- W3["Request Procedure"] --- W4["🚀 Query Execution<br/>💰💰💰 MAIN COST"]
    end
    
    WALMART ===>|"2️⃣ Submit requests"| ADVERTISER
    ADVERTISER ===>|"3️⃣ Return approval status"| WALMART
    
    style ADVERTISER fill:#e6f7ff,stroke:#00A1D9,stroke-width:4px
    style WALMART fill:#0071ce,stroke:#041e42,stroke-width:4px,color:#fff
    style A2 fill:#00A1D9,color:#fff
    style W4 fill:#ffc220,color:#000
```

---

## Request Lifecycle

```mermaid
sequenceDiagram
    autonumber
    participant W as 🛒 Walmart
    participant WW as Walmart Warehouse
    participant AW as Advertiser Warehouse
    participant A as 📺 Advertiser
    
    Note over W,A: SETUP PHASE (One-time)
    W->>WW: Create request procedure
    A->>AW: Create validation tasks
    
    Note over W,A: REQUEST PHASE
    W->>WW: Call request() procedure
    WW->>WW: Build query from template
    WW->>WW: Hash proposed query
    W->>A: Insert request (via share)
    
    Note over W,A: VALIDATION PHASE
    A->>AW: Task detects new request
    AW->>AW: Validate timestamp
    AW->>AW: Validate dimensions
    AW->>AW: Regenerate & compare hash
    A->>A: Log approval status
    
    Note over W,A: EXECUTION PHASE
    W->>W: Check provider_log for approval
    W->>WW: Execute approved query
    WW->>WW: Data Firewall validates hash
    WW->>WW: Join Advertiser + Walmart data
    WW-->>W: Return aggregated results
```

---

## Cost Breakdown by Phase

### Phase 1: Setup (One-time)

| Action | Who Runs | Who Pays | Cost Level |
|--------|----------|----------|------------|
| Create advertiser database, schemas, templates | Advertiser | Advertiser | 💰 Low |
| Create secure views with data firewall | Advertiser | Advertiser | 💰 Low |
| Create share and listing | Advertiser | Advertiser | 💰 Low |
| Mount shared database | Walmart | Walmart | 💰 Low |
| Create request tables and procedures | Walmart | Walmart | 💰 Low |

### Phase 2: Request Submission

| Action | Who Runs | Who Pays | Cost Level |
|--------|----------|----------|------------|
| Call `request()` stored procedure | Walmart | Walmart | 💰 Low |
| Build SQL from Jinja template | Walmart | Walmart | 💰 Low |
| Insert request into shared table | Walmart | Walmart | 💰 Low |

### Phase 3: Validation (Advertiser Tasks)

| Action | Who Runs | Who Pays | Cost Level |
|--------|----------|----------|------------|
| Stream monitors for new requests | Advertiser | Advertiser | 💰 Low |
| Task runs `process_requests()` | Advertiser | Advertiser | 💰💰 Medium |
| Validate timestamp, dimensions | Advertiser | Advertiser | 💰 Low |
| Regenerate query hash | Advertiser | Advertiser | 💰 Low |
| Write to request_log | Advertiser | Advertiser | 💰 Low |

> ⚠️ Advertiser runs **6 parallel tasks** every minute per consumer - this is a recurring cost.

### Phase 4: Query Execution (The Heavy Lifting)

| Action | Who Runs | Who Pays | Cost Level |
|--------|----------|----------|------------|
| Execute approved query | Walmart | Walmart | 💰💰💰 **HIGH** |
| Scan advertiser data (via secure views) | Walmart | Walmart | 💰💰💰 **HIGH** |
| Scan Walmart data | Walmart | Walmart | 💰💰💰 **HIGH** |
| Join and aggregate results | Walmart | Walmart | 💰💰💰 **HIGH** |

---

## Visual Cost Distribution

```mermaid
%%{init: {'theme': 'base', 'themeVariables': { 'pie1': '#0071ce', 'pie2': '#ffc220', 'pie3': '#00A1D9', 'pie4': '#76c8e8', 'pieStrokeColor': '#041e42', 'pieStrokeWidth': '2px'}}}%%
pie showData
    title Who Pays for What?
    "Walmart - Query Execution" : 70
    "Walmart - Setup & Requests" : 10
    "Advertiser - Validation Tasks" : 15
    "Advertiser - Setup" : 5
```

---

## The Data Firewall: How It Works

The Advertiser's data is protected by a **Row Access Policy** that only allows access when:

1. The query hash matches an approved request
2. The request was made by the current account (Walmart)
3. The request has been approved

```mermaid
flowchart TB
    Q[Walmart Query] ==> DF{Data Firewall}
    DF ==>|"✅ Hash matches"| D[(Advertiser Data)]
    DF ==>|"❌ Hash mismatch"| X[No rows returned]
    
    style Q fill:#0071ce,stroke:#333,stroke-width:2px,color:#fff
    style DF fill:#ffc220,stroke:#333,stroke-width:2px,color:#000
    style D fill:#00A1D9,stroke:#333,stroke-width:2px,color:#fff
    style X fill:#c62828,stroke:#333,stroke-width:2px,color:#fff
    
    linkStyle default stroke:#333,stroke-width:3px
```

---

## Key Takeaways

1. **Walmart pays most costs** - They execute the actual queries that scan data
2. **Advertiser pays validation costs** - Lightweight but recurring (tasks run every minute)
3. **Data never moves** - Only queries and results cross account boundaries
4. **Templates control access** - Advertiser defines exactly what queries are allowed
5. **Aggregation enforced** - `HAVING count(*) > 25` prevents individual record exposure

---

*Document generated from DCR v5.5 scripts*
