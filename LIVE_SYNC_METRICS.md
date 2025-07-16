# Live Sync Performance Metrics Overview

## Live Sync Flow Architecture

### 1. FCU (ForkchoiceUpdated) End-to-End Flow

```
on_forkchoice_updated()
├── Pre-validation and state checks
├── Head block validation
├── Canonical chain updates
└── Payload attributes processing (if applicable)
```

### 2. Overall Block Processing Flow (insert_block)

```
insert_block()
├── Start: total_start = Instant::now()
├── Call insert_block_inner()
│   ├── Block validation and consensus checks
│   ├── State provider building
│   ├── Trie input computation
│   ├── Proof generation (if applicable)
│   ├── Block execution
│   ├── State root computation
│   └── Block insertion into tree
├── End: live_sync_block_total_duration.record(total_start.elapsed())
└── Return result
```

### 3. Block Insertion Flow (insert_block_inner)

```
insert_block_inner()
├── Start: start = Instant::now()
├── 1. Trie Input Computation Phase
│   ├── trie_input_start = Instant::now()
│   ├── compute_trie_input() (includes database reads)
│   ├── Database state provider building
│   ├── In-memory block collection and filtering
│   └── trie_input_duration.record(trie_input_elapsed)
├── 2. Proof Generation Phase (Optional)
│   ├── proof_gen_start = Instant::now()
│   ├── spawn() or spawn_cache_exclusive()
│   ├── Trie proof generation for parallel state root
│   └── live_sync_proof_generation_duration.record()
├── 3. Block Execution Phase
│   ├── execution_start = Instant::now()
│   ├── execute_block() with EVM execution
│   ├── Transaction processing and state changes
│   └── live_sync_block_execution_duration.record()
├── 4. State Root Computation Phase
│   ├── Parallel path: live_sync_state_root_parallel_duration
│   ├── Serial fallback: live_sync_state_root_serial_duration
│   ├── record_state_root() method handles both paths
│   └── Hash computation: live_sync_hash_computation_duration
├── 5. Block Insertion Phase
│   ├── insert_start = Instant::now()
│   ├── memory_ops_start = Instant::now()
│   ├── tree_state.insert_executed()
│   ├── live_sync_memory_ops_duration.record()
│   └── live_sync_block_insert_duration.record()
└── End: Return InsertPayloadOk
```

### 3. State Root Computation Flow (record_state_root)

```
record_state_root()
├── Storage tries count: state_root_storage_tries_updated_total
├── Parallel vs Serial decision:
│   ├── If is_parallel: live_sync_state_root_parallel_duration
│   └── If serial: live_sync_state_root_serial_duration
└── Hash computation: live_sync_hash_computation_duration
```

## Key Performance Indicators (KPIs)

### 1. Core Flow Metrics

-   **`live_sync_block_total_duration`** - Overall block processing time (from start to finish)
-   **`live_sync_block_execution_duration`** - Block execution time
-   **`live_sync_state_root_parallel_duration`** - Parallel state root computation time
-   **`live_sync_state_root_serial_duration`** - Serial state root computation time (fallback)
-   **`live_sync_block_insert_duration`** - Block insertion time

### 2. Performance Analysis Metrics

-   **`trie_input_duration`** - Trie input computation time (includes database reads)
-   **`live_sync_hash_computation_duration`** - Hash computation time
-   **`live_sync_proof_generation_duration`** - Trie proof generation time
-   **`live_sync_memory_ops_duration`** - Memory operations overhead time

## Key Monitoring Points

### 1. Trie Input Computation (`trie_input_duration`)

-   **Location**: Trie input computation phase in `insert_block_inner()`
-   **Purpose**: Monitor database IO performance and trie input preparation
-   **Process**: 
  - Database state provider building
  - In-memory block collection and filtering
  - Trie input preparation for state root computation
-   **Optimization**: Caching strategies, query optimization
-   **Note**: This metric includes database read operations that were previously tracked separately

### 2. Hash Computation (`live_sync_hash_computation_duration`)

-   **Location**: State root computation phase (both parallel and serial paths)
-   **Purpose**: Monitor intensive hash calculation performance
-   **Process**: 
  - State root computation involves extensive hash operations
  - Parallel and serial paths both contribute to this metric
-   **Optimization**: Parallelization, hardware acceleration

### 3. Proof Generation (`live_sync_proof_generation_duration`)

-   **Location**: Parallel state root task startup in `insert_block_inner()`
-   **Purpose**: Monitor Trie proof generation overhead
-   **Process**: 
  - Trie proof generation for parallel state root computation
  - Only recorded when using state root tasks
-   **Optimization**: Proof caching, batch processing

### 4. Memory Operations (`live_sync_memory_ops_duration`)

-   **Location**: Tree state insertion operation (subset of block insertion)
-   **Purpose**: Monitor memory allocation/deallocation overhead during tree state updates
-   **Process**: 
  - Memory operations during `tree_state.insert_executed()`
  - Object creation and management for executed blocks
-   **Optimization**: Memory pools, object reuse