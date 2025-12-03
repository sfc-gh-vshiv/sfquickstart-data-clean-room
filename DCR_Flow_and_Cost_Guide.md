# Snowflake Data Clean Room: Execution Flow & Cost Guide

A concise guide explaining where queries run and who pays in a Snowflake Data Clean Room.

---

## Quick Summary

| Party | What They Do | Pays For |
|-------|-------------|----------|
| **Provider** | Hosts data, validates requests, approves queries | Tasks, request processing, template validation |
| **Consumer** | Submits requests, executes approved queries | Query execution, data scanning, result generation |

> **Key Insight**: The Consumer pays for the heavy lifting (query execution), while the Provider pays for lightweight validation tasks.

---

## Architecture Overview

```mermaid
%%{init: {'theme': 'base', 'themeVariables': {'lineColor': '#000000', 'primaryColor': '#e7f5ff'}}}%%
flowchart TB
    subgraph PROVIDER["🏢 PROVIDER ACCOUNT"]
        P1["Provider Data"] --- P2["🛡️ Data Firewall"] --- P3["Templates"] --- P4["Request Log"]
    end
    
    PROVIDER ===>|"1️⃣ Share templates & protected views"| CONSUMER
    
    subgraph CONSUMER["🏪 CONSUMER ACCOUNT"]
        C1["Mounted Share"] --- C2["Consumer Data"] --- C3["Request Procedure"] --- C4["🚀 Query Execution<br/>💰💰💰 MAIN COST"]
    end
    
    CONSUMER ===>|"2️⃣ Submit requests"| PROVIDER
    PROVIDER ===>|"3️⃣ Return approval status"| CONSUMER
    
    style PROVIDER fill:#e7f5ff,stroke:#1971c2,stroke-width:4px
    style CONSUMER fill:#fff3bf,stroke:#f59f00,stroke-width:4px
    style P2 fill:#4dabf7,color:#fff
    style C4 fill:#ff6b6b,color:#fff
```

---

## Request Lifecycle

```mermaid
sequenceDiagram
    autonumber
    participant C as 👤 Consumer
    participant CW as Consumer Warehouse
    participant PW as Provider Warehouse
    participant P as 🏢 Provider
    
    Note over C,P: SETUP PHASE (One-time)
    C->>CW: Create request procedure
    P->>PW: Create validation tasks
    
    Note over C,P: REQUEST PHASE
    C->>CW: Call request() procedure
    CW->>CW: Build query from template
    CW->>CW: Hash proposed query
    C->>P: Insert request (via share)
    
    Note over C,P: VALIDATION PHASE
    P->>PW: Task detects new request
    PW->>PW: Validate timestamp
    PW->>PW: Validate dimensions
    PW->>PW: Regenerate & compare hash
    P->>P: Log approval status
    
    Note over C,P: EXECUTION PHASE
    C->>C: Check provider_log for approval
    C->>CW: Execute approved query
    CW->>CW: Data Firewall validates hash
    CW->>CW: Join Provider + Consumer data
    CW-->>C: Return aggregated results
```

---

## Cost Breakdown by Phase

### Phase 1: Setup (One-time)

| Action | Who Runs | Who Pays | Cost Level |
|--------|----------|----------|------------|
| Create provider database, schemas, templates | Provider | Provider | 💰 Low |
| Create secure views with data firewall | Provider | Provider | 💰 Low |
| Create share and listing | Provider | Provider | 💰 Low |
| Mount shared database | Consumer | Consumer | 💰 Low |
| Create request tables and procedures | Consumer | Consumer | 💰 Low |

### Phase 2: Request Submission

| Action | Who Runs | Who Pays | Cost Level |
|--------|----------|----------|------------|
| Call `request()` stored procedure | Consumer | Consumer | 💰 Low |
| Build SQL from Jinja template | Consumer | Consumer | 💰 Low |
| Insert request into shared table | Consumer | Consumer | 💰 Low |

### Phase 3: Validation (Provider Tasks)

| Action | Who Runs | Who Pays | Cost Level |
|--------|----------|----------|------------|
| Stream monitors for new requests | Provider | Provider | 💰 Low |
| Task runs `process_requests()` | Provider | Provider | 💰💰 Medium |
| Validate timestamp, dimensions | Provider | Provider | 💰 Low |
| Regenerate query hash | Provider | Provider | 💰 Low |
| Write to request_log | Provider | Provider | 💰 Low |

> ⚠️ Provider runs **6 parallel tasks** every minute per consumer - this is a recurring cost.

### Phase 4: Query Execution (The Heavy Lifting)

| Action | Who Runs | Who Pays | Cost Level |
|--------|----------|----------|------------|
| Execute approved query | Consumer | Consumer | 💰💰💰 **HIGH** |
| Scan provider data (via secure views) | Consumer | Consumer | 💰💰💰 **HIGH** |
| Scan consumer data | Consumer | Consumer | 💰💰💰 **HIGH** |
| Join and aggregate results | Consumer | Consumer | 💰💰💰 **HIGH** |

---

## Visual Cost Distribution

```mermaid
%%{init: {'theme': 'base', 'themeVariables': { 'pie1': '#d12b1f', 'pie2': '#fcba03', 'pie3': '#FFB74D', 'pie4': '#94a14a', 'pieStrokeColor': '#333', 'pieStrokeWidth': '2px'}}}%%
pie showData
    title Who Pays for What?
    "Consumer - Query Execution" : 70
    "Provider - Validation Tasks" : 15
    "Consumer - Setup & Requests" : 10
    "Provider - Setup" : 5
```

---

## The Data Firewall: How It Works

The Provider's data is protected by a **Row Access Policy** that only allows access when:

1. The query hash matches an approved request
2. The request was made by the current account
3. The request has been approved

```mermaid
flowchart TB
    Q[Consumer Query] ==> DF{Data Firewall}
    DF ==>|"✅ Hash matches"| D[(Provider Data)]
    DF ==>|"❌ Hash mismatch"| X[No rows returned]
    
    style Q fill:#fff3bf,stroke:#333,stroke-width:2px,color:#000
    style DF fill:#ffcdd2,stroke:#333,stroke-width:2px,color:#000
    style D fill:#c8e6c9,stroke:#333,stroke-width:2px,color:#000
    style X fill:#c62828,stroke:#333,stroke-width:2px,color:#fff
    
    linkStyle default stroke:#333,stroke-width:3px
```

---

## Key Takeaways for Customers

1. **Consumer pays most costs** - They execute the actual queries that scan data
2. **Provider pays validation costs** - Lightweight but recurring (tasks run every minute)
3. **Data never moves** - Only queries and results cross account boundaries
4. **Templates control access** - Provider defines exactly what queries are allowed
5. **Aggregation enforced** - `HAVING count(*) > 25` prevents individual record exposure

---

*Document generated from DCR v5.5 scripts*

