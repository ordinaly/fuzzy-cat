# Fuzzy Cat Architecture

## System Overview

Fuzzy Cat is a distributed system for recording, aggregating, and analyzing deliberative processes at scale. The system is designed around these principles:

- **Decentralization**: No single point of control
- **Transparency**: All operations are auditable
- **Immutability**: Records cannot be changed retroactively
- **Privacy**: Individual dignity is protected by design
- **Scalability**: Can handle millions of triangles across multiple hierarchies

---

## High-Level Architecture

```
┌──────────────────────────────────────────────────────────────┐
│                     User Interface Layer                       │
│                    (React Frontend)                            │
│  Triangle Formation | Deliberation | Visualization | Analysis │
└────────────────────────┬─────────────────────────────────────┘
                         │ HTTP/WebSocket
                         ↓
┌──────────────────────────────────────────────────────────────┐
│                      API Layer                                 │
│                   (FastAPI Backend)                            │
│  /triangles  /meta-triangles  /analysis  /blockchain  /query  │
└────────────────────────┬─────────────────────────────────────┘
                         │
        ┌────────────────┼────────────────┐
        ↓                ↓                ↓
   ┌─────────┐    ┌────────────┐    ┌─────────────┐
   │Aggreg.  │    │Blockchain  │    │Analysis     │
   │Engine   │    │Interface   │    │Engine       │
   │(Python) │    │(Web3.py)   │    │(NumPy/Scipy)│
   └────┬────┘    └────┬───────┘    └─────────────┘
        │              │
        │              ↓
        │         ┌──────────────┐
        │         │  Ethereum    │
        │         │  Blockchain  │
        │         │  Mainnet/    │
        │         │  Testnet     │
        │         └──────────────┘
        │
        ↓
   ┌──────────────┐
   │ PostgreSQL   │
   │ (Metadata)   │
   └──────────────┘
```

---

## Component Details

### 1. Frontend (React/D3.js)

**Purpose**: User interface for triangle formation, deliberation, and visualization

**Key Components**:
- `TriangleForm.jsx` — Interface for creating triangles and recording participant metadata
- `DeliberationView.jsx` — Real-time interface for recording consensus as it emerges
- `DashboardView.jsx` — Overview of all triangles, meta-triangles, etc.
- `PatternVisualization.jsx` — Interactive D3.js visualizations of patterns at each level
- `MetadataFilter.jsx` — Filter and search by demographic characteristics
- `QueryBuilder.jsx` — Build custom analyses

**Data Flow**:
1. User creates a triangle with 3 participants
2. Participants record their initial positions (probability distributions)
3. Deliberation happens (audio/notes optional, not required)
4. Participants record final consensus output
5. Frontend sends data to backend API

**Technology Stack**:
- React 18+
- D3.js for visualizations
- Material-UI for components
- Redux for state management
- Web3.js for blockchain interaction

---

### 2. Backend API (Python/FastAPI)

**Purpose**: Process requests, manage aggregation, interface with blockchain

**Key Endpoints**:

```
POST /api/triangles
  Create a new triangle
  Input: {metadata, initial_positions, question}
  Output: {triangle_id, status}

GET /api/triangles/{id}
  Retrieve triangle details
  Output: {id, metadata, input_distribution, output_distribution, confidence}

POST /api/meta-triangles
  Aggregate three triangles into a meta-triangle
  Input: {triangle_ids: [id1, id2, id3]}
  Output: {meta_id, aggregated_output, confidence}

GET /api/query
  Complex analysis queries
  Input: {level, filters, metric}
  Output: Statistical results

GET /api/blockchain/{record_hash}
  Retrieve blockchain record
  Output: Immutable record with signature verification
```

**Key Services**:

1. **Triangle Service**: CRUD operations on triangles
2. **Aggregation Service**: Consensus algorithm implementation
3. **Blockchain Service**: Record submission and verification
4. **Analysis Service**: Statistical computations
5. **Query Service**: Complex multi-triangle queries

**Technology Stack**:
- FastAPI (async HTTP framework)
- SQLAlchemy (database ORM)
- Pydantic (data validation)
- NumPy/SciPy (numerical computing)
- Web3.py (Ethereum interaction)

---

### 3. Aggregation Engine

**Purpose**: Compute consensus at each level using deterministic algorithm

**Algorithm**:

For a triangle with three inputs $\mathbf{p}_{n-1,1}, \mathbf{p}_{n-1,2}, \mathbf{p}_{n-1,3}$:

```python
def aggregate(p1: Distribution, p2: Distribution, p3: Distribution) -> Triangle:
    # Compute mean
    consensus = (p1 + p2 + p3) / 3
    
    # Compute confidence (alignment)
    variance = (1/3) * sum([
        ||p1 - consensus||^2,
        ||p2 - consensus||^2,
        ||p3 - consensus||^2
    ])
    confidence = 1 - variance
    
    return Triangle(
        output=consensus,
        confidence=confidence,
        diversity_reduction=initial_variance - variance
    )
```

**Key Properties**:
- **Deterministic**: Same input always produces same output
- **Symmetric**: Order of inputs doesn't matter
- **Recursive**: Works at any level (individuals → triangles → meta-triangles → ...)
- **Verifiable**: Can be run independently by anyone with the data

**Implementation**:
- Located in `backend/aggregation/`
- Functions: `aggregate_triangle()`, `aggregate_meta()`, `compute_confidence()`
- Tests in `tests/test_aggregation.py`

---

### 4. Random Assignment Engine

**Purpose**: Assign people to triangles such that de-anonymization becomes computationally intractable

**Design Philosophy**:

Instead of relying on temporal separation or policy enforcement, Fuzzy Cat makes de-anonymization NP-hard through intelligent random assignment with tight diversity constraints.

**Diversity Constraints**:

```python
class TriangleConstraints:
    """Constraints that make valid assignments sparse, de-anonymization hard"""
    
    def expertise_balance(self, person_a, person_b, person_c):
        """Each triangle must have: 1 expert, 1 intermediate, 1 novice on topic"""
        expertise_levels = [p.expertise_level for p in [person_a, person_b, person_c]]
        return sorted(expertise_levels) == ['expert', 'intermediate', 'novice']
    
    def geographic_diversity(self, person_a, person_b, person_c):
        """Must have at least 2 different location types"""
        locations = {p.location_type for p in [person_a, person_b, person_c]}
        return len(locations) >= 2
    
    def demographic_balance(self, person_a, person_b, person_c):
        """Age spread (20+ years), professional diversity, education mix"""
        ages = [p.age for p in [person_a, person_b, person_c]]
        age_span = max(ages) - min(ages)
        
        professions = {p.profession for p in [person_a, person_b, person_c]}
        educations = {p.education for p in [person_a, person_b, person_c]}
        
        return age_span >= 20 and len(professions) >= 2 and len(educations) >= 2
    
    def temporal_constraint(self, person_a, person_b, person_c):
        """No two people who deliberated together in last 30 days"""
        pairs = [(person_a, person_b), (person_a, person_c), (person_b, person_c)]
        for p1, p2 in pairs:
            days_since_last = days_since_deliberated_together(p1, p2)
            if days_since_last < 30:
                return False
        return True
    
    def is_valid_triangle(self, person_a, person_b, person_c):
        """All constraints must be satisfied"""
        return all([
            self.expertise_balance(person_a, person_b, person_c),
            self.geographic_diversity(person_a, person_b, person_c),
            self.demographic_balance(person_a, person_b, person_c),
            self.temporal_constraint(person_a, person_b, person_c),
        ])
```

**Assignment Algorithm**:

```python
def assign_people_to_triangles(people, num_triangles, constraints):
    """
    Random assignment with backtracking to satisfy constraints.
    Creates computationally hard de-anonymization problem.
    """
    assignment = {}
    available = list(people)
    
    for triangle_id in range(num_triangles):
        # Try random combinations until one satisfies all constraints
        max_attempts = 1000
        attempts = 0
        
        while attempts < max_attempts:
            # Random sample of 3 people from available pool
            candidates = random.sample(available, 3)
            
            # Check if this trio satisfies all constraints
            if constraints.is_valid_triangle(*candidates):
                assignment[triangle_id] = candidates
                
                # Remove from available (they've been assigned)
                for person in candidates:
                    available.remove(person)
                
                break
            
            attempts += 1
        
        if attempts == max_attempts:
            # Failed to find valid assignment - backtrack and restart
            # This can happen if constraints are too tight
            return assign_people_to_triangles(people, num_triangles, constraints)
    
    return assignment
```

**Why This Creates NP-Hardness**:

The de-anonymization problem becomes:

```
Given:
  - Set P of N people with characteristics
  - Set T of M observed triangle outputs
  - Set C of diversity constraints
  
Find: An assignment X of people to triangles such that:
  1. Each constraint in C is satisfied
  2. Each assignment produces the observed output
  3. Person p is in triangle t
```

This is a **Constraint Satisfaction Problem (CSP)**, which is NP-complete.

**Attack Complexity**:

| Scenario | Complexity | Feasibility |
|----------|---|---|
| Naive (try all assignments) | O(N choose 3^M) | Infeasible for N > 100 |
| With constraint propagation | Still exponential | Infeasible for N > 1000 |
| With tight constraints | NP-hard worst case | Computationally intractable |

**Practical Security**:

For a system with:
- N = 1,000 people
- M = 50 triangles
- C = 4 tight constraints (expertise, geography, demographics, temporal)

De-anonymization requires solving ~2^k CSP instances where k > 50.

This is **computationally infeasible** even with modern computing.

**Key Insight**: Security doesn't depend on hiding data. Even if attacker has everything (outputs, metadata, constraint definitions), they cannot efficiently recover assignments due to computational hardness.

---

### 5. Blockchain Interface

**Purpose**: Record all triangle decisions immutably on Ethereum

**Smart Contracts**:

#### TriangleRegistry.sol
Stores triangle records with signatures

```solidity
struct TriangleRecord {
    bytes32 id;
    uint256 timestamp;
    bytes32 metadataHash;      // Hash of aggregated metadata
    bytes32 inputHash;         // Hash of input distribution
    bytes32 outputHash;        // Hash of output distribution
    uint8 confidence;          // Confidence as percentage
    bytes[] signatures;        // Signatures from witnesses
}

function recordTriangle(TriangleRecord record) public;
function getTriangle(bytes32 id) public view returns (TriangleRecord);
function verifyTriangle(bytes32 id) public view returns (bool);
```

#### MetaAggregationRegistry.sol
Stores aggregation records

```solidity
struct MetaRecord {
    bytes32 id;
    bytes32[] triangle_ids;    // References to three constituent triangles
    bytes32 outputHash;
    uint8 confidence;
}

function recordAggregation(MetaRecord record) public;
```

**Data Storage Strategy**:
- Immutable record hash stored on-chain (~32 bytes per triangle)
- Full record stored on IPFS (decentralized storage)
- Ethereum stores reference hash, allowing verification without storage bloat

**Cost Optimization**:
- ~$0.60 per triangle recorded (at 20 Gwei gas)
- ~$213K/year for 1,000-person organization (1M transactions)
- Can be reduced with L2 solutions (Optimism, Arbitrum)

**Network Options**:
- Ethereum Testnet (Sepolia/Goerli) for development
- Ethereum Mainnet for production
- Eventually L2s for cost reduction

---

### 6. Analysis Engine

**Purpose**: Statistical analysis of recorded triangles

**Key Analyses**:

1. **Demographic Correlation**
   ```python
   def correlate_demographics(triangles, metric):
       """Which demographics correlate with positions?"""
       # Returns correlation coefficients for each demographic variable
   ```

2. **Pattern Detection**
   ```python
   def detect_patterns(triangles, level):
       """What patterns are visible at this level?"""
       # Identifies clusters, gradients, anomalies
   ```

3. **Deliberation Quality**
   ```python
   def deliberation_quality(triangle):
       """How much did deliberation change consensus?"""
       # Measures: diversity_reduction, confidence_increase
   ```

4. **Hierarchy Analysis**
   ```python
   def hierarchy_analysis(mega_triangles):
       """How does diversity change across levels?"""
       # Variance at each level, convergence rate
   ```

**Technology Stack**:
- NumPy for numerical operations
- SciPy for statistical functions
- Pandas for data manipulation
- Scikit-learn for machine learning (if needed)
- Matplotlib/Seaborn for plotting

---

### 7. Security and Privacy by Design

The Random Assignment Engine combined with irreversible aggregation and blockchain immutability creates a security model based on mathematics, not secrets.

**Security Properties**:
- **De-anonymization hardness**: NP-hard to match individuals to triangles
- **Data irreversibility**: Cannot reverse aggregate back to individuals
- **Immutability**: Blockchain prevents retroactive changes
- **Transparency**: All processes are auditable
- **Robustness**: Security survives data breaches due to computational hardness

See **ETHICS.md** for detailed analysis of security model and risk mitigations.

---

## Data Flow: Complete Example

**Scenario**: 27 people create 9 triangles (level 1), which aggregate to 3 meta-triangles (level 2).

### Step 1: Random Assignment to Triangle

Before any deliberation happens, the system must assign three people to form a triangle.

```
Assignment Engine
  ↓
Selects 3 people from available pool
  ↓
Checks diversity constraints:
  - Expertise: One expert, one intermediate, one novice on topic
  - Geography: Mixed locations (urban/rural/suburban)
  - Demographics: Age range (20+ years), profession diversity, education mix
  - Temporal: Haven't deliberated together in last 30 days
  ↓
If constraints satisfied: Assign to triangle
If constraints not satisfied: Try another random triple
  ↓
Record assignment metadata (aggregated, never individual identities)
```

Result: **Random assignment that makes de-anonymization NP-hard**

### Step 2: Individual Participation
```
User Interface (React)
  ↓
Person A, B, C fill out form:
  - Metadata (age, location, education, background)
  - Question: "Climate vs development priority?"
  - Positions: A=70%, B=50%, C=30%
  ↓
Frontend validates data
  ↓
POST /api/triangles
```

### Step 3: Backend Processing
```
FastAPI receives request
  ↓
Triangle Service validates metadata (no personal identifiers)
  ↓
Aggregation Engine computes:
  - Consensus: (70+50+30)/3 = 50%
  - Confidence: 1 - (variance of [70,50,30]) = 0.85
  ↓
Database stores:
  - Triangle metadata (aggregated: [29,45,51] ages)
  - Output distribution (50% consensus, 0.85 confidence)
  ↓
Blockchain Service prepares record:
  - Hash the output
  - Sign with witness key
  ↓
Submit to Ethereum contract
```

### Step 4: Blockchain Recording
```
Smart contract TriangleRegistry receives transaction
  ↓
Verifies signature
  ↓
Stores record hash on-chain:
  {
    triangle_id: 0x1a2b...,
    output_hash: 0x3c4d...,
    confidence: 85,
    timestamp: 1234567890
  }
  ↓
Emits event: TriangleRecorded(triangle_id, output_hash)
  ↓
Transaction confirmed on blockchain (12 seconds to 1 minute)
```

### Step 5: Aggregation to Meta-Triangle
```
Aggregation Engine receives three triangles:
  - Triangle 1: 60% (confidence 0.88)
  - Triangle 2: 45% (confidence 0.90)
  - Triangle 3: 50% (confidence 0.85)
  ↓
Computes meta-consensus:
  - Output: (60+45+50)/3 = 51.7%
  - Confidence: 1 - variance = 0.91
  ↓
Records meta-triangle on blockchain
  ↓
Process repeats up the hierarchy
```

### Step 6: Analysis and Visualization
```
Frontend queries: GET /api/query?level=2&filter=age>40
  ↓
Analysis Engine computes:
  - "Triangles with avg age > 40 prefer 55% climate"
  - "Mixed-age triangles prefer 50%"
  - "Age difference explains 12% of variance"
  ↓
Returns statistics
  ↓
Frontend renders D3 visualization
  ↓
User sees: Interactive chart showing patterns
```

---

## Scalability Considerations

### Horizontal Scaling

**Triangle Recording**:
- Each triangle is independent
- Can record millions in parallel
- API runs on multiple servers behind load balancer
- Database optimized with indices on triangle_id, timestamp

**Blockchain**:
- Ethereum mainnet: ~12 transactions/second
- Use batching: aggregate multiple triangles, record batch hash
- Alternative: Use L2 solutions (Optimism, Arbitrum) for 1000x throughput

**Storage**:
- IPFS for full records (distributed)
- PostgreSQL for metadata indexes
- Can shard by triangle_id or timestamp

### Vertical Scaling

**Level Limits**:
- Level 1: 3-3,000 people (3 to ~3000 triangles)
- Level 2: Requires 3+ meta-triangles = 9+ level-1 triangles = 27+ people
- Level 3: 81+ people
- Level $n$: $3^n$ people minimum

**Computational Complexity**:
- Per triangle: O(d) where d = dimension of position vector
- Aggregation: O(1) per level (constant time to compute mean)
- Analysis: O(n²) for correlations, O(n log n) for sorting

**Network**:
- Blockchain: Limited by Ethereum throughput (~12-15 tx/sec mainnet)
- Solution: Batch records, use L2s

---

## Security Considerations

### Privacy
- Individual positions never recorded
- Metadata aggregated (ages [25,35,45], not linked to people)
- No way to reverse engineer individuals from triangle outputs (mathematically impossible—underdetermined system)

### Integrity
- Blockchain provides immutability
- Smart contracts verify signatures before recording
- Cryptographic hashing prevents tampering

### Availability
- Decentralized blockchain ensures availability
- IPFS ensures distributed storage
- PostgreSQL replicates for redundancy

---

## Development Workflow

### Local Development
```bash
# Terminal 1: Backend
cd backend
python -m venv venv
source venv/bin/activate
pip install -r requirements.txt
python main.py  # Runs on http://localhost:8000

# Terminal 2: Frontend
cd frontend
npm install
npm start  # Runs on http://localhost:3000

# Terminal 3: Local Ethereum (optional)
docker run -p 8545:8545 trufflesuite/ganache-cli
```

### Testing
```bash
# Backend tests
pytest tests/test_aggregation.py -v
pytest tests/test_blockchain.py -v
pytest tests/test_api.py -v

# Frontend tests
cd frontend && npm test

# Smart contract tests
cd contracts && npm test
```

### Deployment
```bash
# Staging (Ethereum Sepolia)
docker-compose -f docker-compose.staging.yml up

# Production (Ethereum Mainnet)
docker-compose -f docker-compose.prod.yml up
```

---

## Monitoring and Observability

### Metrics
- Triangles recorded per hour
- Average confidence by level
- Blockchain transaction costs
- API response times
- Database query performance

### Logging
- All aggregation operations logged
- Blockchain transactions tracked
- User actions audited
- Errors flagged for investigation

### Alerts
- Failed blockchain submissions
- Inconsistent aggregations
- Database performance degradation
- API timeout

---

## Future Optimizations

### L2 Deployment
Migrate to Optimism or Arbitrum for 100x cost reduction and 1000x throughput improvement

### Off-Chain Aggregation
Compute aggregation off-chain, verify on-chain using merkle proofs

### Sharding
Partition triangles by question or time period for parallel processing

### ML Integration
Use machine learning to predict deliberation outcomes based on triangle composition

---

## Conclusion

Fuzzy Cat's architecture is designed around these principles:
1. **Decentralization** — No single point of control
2. **Transparency** — Everything is auditable
3. **Privacy** — Dignity protected by design
4. **Scalability** — Can handle millions
5. **Immutability** — History preserved forever
6. **Simplicity** — Easy to understand and verify

The system is modular, allowing independent components to be upgraded or replaced without affecting the whole.
