# 🌐 FraudNexus — Financial Crime & Synthetic Identity Graph Explorer

> **Wexa AI — Software Engineer Take-Home Assignment**  
> Backed by **CognoDB Cloud** (openCypher over Bolt 5.0 protocol) using official `neo4j-driver`.

[![CognoDB Cloud](https://img.shields.io/badge/Database-CognoDB%20Cloud%20(Bolt%205.0)-3b82f6?style=for-the-badge&logo=neo4j)](https://console.cognodb.com)
[![Next.js 14](https://img.shields.io/badge/Framework-Next.js%2014-000000?style=for-the-badge&logo=nextdotjs)](https://nextjs.org)
[![TypeScript](https://img.shields.io/badge/Language-TypeScript-3178C6?style=for-the-badge&logo=typescript)](https://www.typescriptlang.org)
[![Tailwind CSS](https://img.shields.io/badge/Styling-Tailwind%20CSS-06B6D4?style=for-the-badge&logo=tailwindcss)](https://tailwindcss.com)

---

## 📌 Executive Summary

**FraudNexus** is a full-stack, highly interactive Web Application backed by **CognoDB** (a managed graph database speaking openCypher over the Bolt protocol). 

In financial fraud detection, critical patterns are hidden within **relationships and connections**—such as multi-hop circular money laundering rings (*smurfing*), synthetic identities sharing devices or IP addresses, and nested offshore shell companies hiding the Ultimate Beneficial Owner (UBO).

FraudNexus allows compliance officers, fraud analysts, and engineers to visualize graph topologies in real time, run parameterized openCypher queries, calculate contagion blast radii, and inject live transactions into CognoDB Cloud.

---

## 🧠 Why a Graph Database? (Relational vs. Graph)

When detecting financial fraud, the most vital questions are about **paths, networks, and depth**:

| Feature / Query Pattern | Relational Database (SQL) | CognoDB Graph Database (openCypher) |
| :--- | :--- | :--- |
| **Multi-Hop Circular Transfer Loops** (A → B → C → D → A) | Requires nested SQL `JOIN`s or recursive `WITH RECURSIVE` CTEs. Query complexity grows exponentially $O(N^k)$, causing query timeouts. | `(a:Account)-[:TRANSFERRED_TO*3..6]->(a)` <br/>Traverses physical pointers in constant time $O(1)$ per hop via **Index-Free Adjacency**. |
| **Synthetic Identity Resolution** (Shared Devices/IPs) | Requires 5-table junction joins (`Person`, `Person_Device`, `Device`, `Person_Account`, `Account`) with expensive `GROUP BY`s. | `(p1)-[:USED_DEVICE]->(d)<-[:USED_DEVICE]-(p2)` <br/>Traverses heterogeneous connections bi-directionally without junction tables. |
| **Offshore UBO Traversal** (Dynamic Shell Company Depth) | Dynamic depth (1-10 holding tiers) breaks fixed SQL schema assumptions. | `(p:Person)-[:BENEFICIAL_OWNER_OF*1..5]->(c:Company)` <br/>Effortlessly tracks dynamic ownership depth. |
| **Shortest Path Blast Radius** | Requires exporting table datasets to application memory and implementing Dijkstra's algorithm manually. | `shortestPath((start)-[*..6]-(target))` <br/>Native database-level breadth-first graph traversal algorithm. |

---

## 📐 Graph Data Model

The application models a multi-entity financial network with **4 labeled nodes** and **4 typed relationships**:

```mermaid
graph TD
    Person["(:Person)<br/>fullName, ssn, status"]
    Company["(:Company)<br/>companyName, regNumber, isShell"]
    Account["(:Account)<br/>accountNumber, balance, bank, riskScore"]
    Device["(:Device)<br/>fingerprint, ipAddress, isTor"]

    Person -->|OWNS| Account
    Person -->|USED_DEVICE| Device
    Person -->|BENEFICIAL_OWNER_OF {percentage}| Company
    Company -->|BENEFICIAL_OWNER_OF {percentage}| Company
    Account -->|TRANSFERRED_TO {amount, timestamp, riskFlag}| Account
```

### ASCII Graph Model Schema

```text
    ┌────────────────┐                  ┌────────────────┐
    │    :Person     │───[:USED_DEVICE]─▶│    :Device     │
    │ ssn, fullName  │                  │ fingerprint, IP│
    └───────┬────────┘                  └────────────────┘
            │
      [:OWNS│
            ▼
    ┌────────────────┐  ──[:TRANSFERRED_TO]──▶ ┌────────────────┐
    │    :Account    │                         │    :Account    │
    │ balance, bank  │ ◀─[:TRANSFERRED_TO]──── │ balance, bank  │
    └────────────────┘                         └────────────────┘
```

---

## ⚡ Main openCypher Queries Explained

All queries execute via the official `neo4j-driver` using **parameterized inputs** to prevent Cypher injection and maximize database query plan caching.

### 1. Circular Money Laundering Ring Detection (3-6 Hop Cycles)
Finds accounts transferring funds in circular loops where money returns to the originating account.
```cypher
MATCH path = (a:Account {id: $startAccount})-[:TRANSFERRED_TO*3..6]->(a)
RETURN path, 
       reduce(total = 0, tx IN relationships(path) | total + tx.amount) AS totalVolume,
       length(path) AS loopLength
ORDER BY totalVolume DESC
LIMIT 10;
```

### 2. Synthetic Identity Resolution (Shared Device Fingerprint)
Identifies distinct individuals sharing the exact same hardware fingerprint or Tor exit node.
```cypher
MATCH (p1:Person)-[:USED_DEVICE]->(d:Device)<-[:USED_DEVICE]-(p2:Person)
WHERE p1.id <> p2.id
OPTIONAL MATCH (p1)-[:OWNS]->(a1:Account)
OPTIONAL MATCH (p2)-[:OWNS]->(a2:Account)
RETURN p1, p2, d, a1, a2
LIMIT 25;
```

### 3. Ultimate Beneficial Owner (UBO) Offshore Shell Traversal
Traces corporate equity chains through dynamic shell company layers to sanctioned individuals.
```cypher
MATCH path = (p:Person)-[:BENEFICIAL_OWNER_OF*1..5]->(c:Company)
WHERE p.status = $status OR c.isShell = true
RETURN path, p, c
ORDER BY length(path) DESC;
```

### 4. Account Risk Blast Radius & Shortest Path
Calculates contagion distance between a compromised account and target institutions.
```cypher
MATCH (start:Account {id: $startId}), (target:Account {id: $targetId})
MATCH path = shortestPath((start)-[:TRANSFERRED_TO|OWNS|USED_DEVICE*..6]-(target))
RETURN path, length(path) AS distance;
```

---

## 🚀 Setup & Installation Instructions

### Prerequisites
- Node.js **18.x** or **20.x**
- npm / pnpm / yarn

---

### Step 1: Set up CognoDB Cloud Instance

1. Go to **[https://console.cognodb.com/signup](https://console.cognodb.com/signup)** and create a free account (No credit card required).
2. Click **Create Instance** -> select the free **c0** tier and a region. Provisioning completes in ~30 seconds.
3. Save your generated connection details:
   - **Connection URI**: `bolt+s://<instance-id>.databases.cognodb.cloud`
   - **Username**: `cognodb`
   - **Password**: *(your saved generated password)*

---

### Step 2: Clone Repository & Configure Environment Variables

```bash
git clone https://github.com/your-username/fraudnexus.git
cd fraudnexus
npm install
```

Create a `.env` file in the root directory:

```env
# CognoDB Cloud Connection Credentials
COGNODB_URI=bolt+s://<your-instance-id>.databases.cognodb.cloud
COGNODB_USER=cognodb
COGNODB_PASSWORD=your-saved-cognodb-password

# Set to 'false' to connect to live CognoDB Cloud instance
FORCE_MOCK_MODE=false
```

---

### Step 3: Run Database Seed Script

Populate your CognoDB Cloud instance with initial realistic financial fraud networks (50+ nodes, 100+ relationships across smurfing rings, synthetic IDs, and shell companies):

```bash
npm run seed
```

---

### Step 4: Run Web Application Locally

```bash
npm run dev
```

Open **[http://localhost:3000](http://localhost:3000)** in your browser to explore FraudNexus!

---

## 🎨 UI/UX Features & Interactive Engineering

- ⚛️ **HTML5 Physics Force Graph Canvas**: Drag nodes, zoom/pan, center view, and toggle particle animation representing real-time money flow.
- 🔴 **Glowing Risk Score Gauges**: Automatic color-coded glow for critical risk nodes (90+ score = Red, 75+ = Orange, 40+ = Yellow).
- 💻 **Live openCypher Query Console**: View query parameters, execution latency in milliseconds (`12ms`), and edit/run custom Cypher queries live.
- 🛡️ **Account Risk Blast Radius Calculator**: Compute shortest path contagion distance across multi-hop graphs.
- ➕ **Live Graph Transaction Injector**: Inject new transfer relationships live into CognoDB Cloud.
- 🔌 **Graceful Database Error Handling & Sandbox Mock Toggle**: If CognoDB credentials are omitted or the cloud instance is offline, the app smoothly transitions to an interactive **Mock Sandbox Mode**, allowing 100% UI and query exploration.

---

## 🛠️ Project Structure

```text
fraudnexus/
├── app/
│   ├── api/
│   │   ├── health/          # Check CognoDB connection health & latency
│   │   ├── graph/nodes/     # Graph topology & label/risk search filter endpoint
│   │   ├── graph/query/     # Parameterized Cypher query executor
│   │   └── graph/transaction/# Live relationship injection endpoint
│   ├── globals.css          # Dark cyber theme CSS styles & animations
│   ├── layout.tsx           # Next.js root layout & metadata
│   └── page.tsx             # Main dashboard page
├── components/
│   ├── Navbar.tsx           # Header bar with live CognoDB health ping & mode toggle
│   ├── Sidebar.tsx          # Graph filter drawer & Cypher scenario presets
│   ├── GraphCanvas.tsx      # HTML5 Canvas 2D physics force graph renderer
│   ├── NodeInspector.tsx    # Slide-over drawer for entity properties & 1-hop edges
│   ├── CypherConsole.tsx    # Terminal console showing Cypher, params & execution time
│   ├── WhyGraphModal.tsx    # Interactive relational vs graph comparison modal
│   ├── BlastRadiusModal.tsx # Shortest path contagion calculator modal
│   └── TransactionModal.tsx # Live transaction edge creator modal
├── lib/
│   ├── db/
│   │   └── cognoDB.ts       # Neo4j driver singleton targeting CognoDB Cloud over Bolt
│   ├── mock/
│   │   └── mockData.ts      # Offline sandbox fallback graph dataset & scenario presets
│   └── types/
│       └── graph.ts         # TypeScript definitions for Nodes, Relationships & Cypher queries
├── scripts/
│   └── seed.ts              # Automated data loader populating CognoDB Cloud
├── .env.example             # Environment variable template
├── package.json
└── README.md

