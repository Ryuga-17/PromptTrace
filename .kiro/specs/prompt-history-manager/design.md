# Design Document: Prompt History Manager

## Overview

The Prompt History Manager is an AI-assisted code editor system that captures, organizes, and preserves the reasoning behind AI-generated code. Instead of treating AI prompts as temporary chat messages, the system treats them as development artifacts, similar to commits or decision records in Git-based workflows.

The core goal is to make AI-assisted development collaborative, explainable, and maintainable, especially for students and open-source projects. The system does not aim to generate better code than existing AI editors—it focuses on what happens after code is generated: tracking why changes were made and how the code evolved over time.

The system observes every prompt used in the code editor, evaluates its intent and impact, and selectively converts meaningful prompts into structured documentation. The architecture follows a pipeline pattern where prompts flow through qualification, classification, verification, grouping, and summarization stages.

Key design principles:
- **Verification over trust**: Use code diffs to verify actual impact
- **Semantic grouping**: Use similarity measures to group related work into tasks
- **Progressive summarization**: Merge refinements into cohesive task descriptions
- **Storage efficiency**: Compress old history while preserving key decisions
- **Context lineage**: Build a prompt tree (DAG) showing refinements and dependencies

## Architecture

The system operates as an integrated pipeline within an AI-enabled code editor. The flow is not strictly linear—classification happens after code generation to verify actual impact, and storage uses a date-based hierarchy by default with vector DB only for search/reordering.

### System Flow

```
┌──────────────────────────────────────────────────────────────┐
│                    AI Code Editor (IDE)                       │
└───────────────────────────┬──────────────────────────────────┘
                            │ User enters prompt
                            ▼
┌─────────────────────────────────────────────────────────────┐
│  1. Prompt Entry & Context Capture                          │
│     - Capture current codebase state (before snapshot)      │
│     - Optional: Attach context from earlier prompts         │
└───────────────────────────┬─────────────────────────────────┘
                            ▼
┌─────────────────────────────────────────────────────────────┐
│  2. LLM Code Generation with Integrated Classification      │
│     - AI generates/modifies code                            │
│     - LLM simultaneously classifies prompt type             │
│     - LLM generates tags based on action type & changes     │
│     - Capture after snapshot                                │
│     - Compare before/after states                           │
│     - Compute diff to verify LLM's classification           │
└───────────────────────────┬─────────────────────────────────┘
                            ▼
┌─────────────────────────────────────────────────────────────┐
│  3. Classification Verification & Override                  │
│     - Verify LLM classification against actual diff         │
│     - User can manually override if needed                  │
│     - Discard if Ephemeral                                  │
└───────────────────────────┬─────────────────────────────────┘
                            ▼
┌─────────────────────────────────────────────────────────────┐
│  4. Prompt Record Creation                                  │
│     - Store LLM-provided classification & tags              │
│     - Link to verified code changes                         │
│     - Reference related prompts                             │
└───────────────────────────┬─────────────────────────────────┘
                            ▼
┌─────────────────────────────────────────────────────────────┐
│  5. Date-Based Hierarchical Storage (Default View)          │
│     - Store in date-based tree: Year/Month/Day/Prompt       │
│     - Like GitHub commit history                            │
│     - Fast chronological access                             │
│     - Also store embedding in vector DB for search          │
└───────────────────────────┬─────────────────────────────────┘
                        ▼
┌─────────────────────────────────────────────────────────────┐
│  6. Prompt Tree (DAG) Construction                          │
│     - Track refinements and dependencies                    │
│     - Enable context reuse                                  │
│     - Prevent contradictory changes                         │
└───────────────────────────┬─────────────────────────────────┘
                            ▼
┌─────────────────────────────────────────────────────────────┐
│  7. Decision Log Storage (if no code change but important)  │
│     - Store rationale without diff                          │
│     - Link to relevant context in tree                      │
└───────────────────────────┴─────────────────────────────────┘
                            │
                ┌───────────┴───────────┐
                ▼                       ▼
┌──────────────────────────┐  ┌──────────────────────────────┐
│  8. Garbage Collection   │  │  9. Documentation Generation │
│     (periodic)           │  │     (on-demand)              │
│  - Compress redundant    │  │  - Default: date-based view  │
│  - Merge refinements     │  │  - Search: use vector DB     │
│  - Preserve decisions    │  │  - Export to Markdown        │
└──────────────────────────┘  └──────────────────────────────┘
```

**Key Architecture Decisions**:

1. **LLM-integrated classification**: The code-generating LLM simultaneously classifies the prompt type and generates tags based on the action type and code changes it's making. This is more efficient than post-hoc classification since the LLM already understands the intent and impact.
2. **Verification step**: After LLM classification, the system verifies it against the actual diff to ensure accuracy
3. **Date-based default storage**: Prompts stored hierarchically by date (Year/Month/Day) for fast chronological access, similar to GitHub's commit history
4. **Vector DB for search only**: Embeddings stored in vector DB, but only used when user searches or reorders—default view is chronological
5. **Parallel final stages**: Garbage collection (periodic) and documentation generation (on-demand) run independently after core pipeline

**Core Components**:

1. **Context Capture**: Snapshots codebase state before prompt execution
2. **LLM Code Generator with Integrated Classifier**: AI generates/modifies code while simultaneously classifying the prompt type and generating relevant tags based on action type and code changes
3. **Diff Analyzer**: Computes actual code changes and verifies LLM's classification
4. **Prompt Record Manager**: Creates and stores structured prompt records with LLM-provided metadata
5. **Hierarchical Storage**: Date-based tree structure for chronological organization
6. **Prompt Tree Builder**: Constructs DAG of refinements and dependencies
7. **Decision Log Manager**: Stores important decisions without code changes
8. **Garbage Collector**: Compresses and cleans history over time (periodic)
9. **Documentation Generator**: Produces human-readable output (on-demand)
10. **Vector DB**: Stores embeddings for semantic search (used on-demand)

## Components and Interfaces

### 1. Context Capture & LLM Code Generation with Classification

**Purpose**: Capture codebase state, generate code, and simultaneously classify the prompt with relevant tags.

**Interface**:
```
capture_context(workspace: FilePath) -> FileSystemSnapshot

generate_code_with_classification(
  prompt: String, 
  context: FileSystemSnapshot
) -> CodeGenerationResult

CodeGenerationResult = {
  modified_snapshot: FileSystemSnapshot,
  classification: PromptType,
  tags: List<String>,
  summary: String
}
```

**Algorithm**:
1. Capture before snapshot of codebase
2. Send prompt to LLM code generator with instructions to:
   - Generate/modify code
   - Classify the prompt type (Actionable/Refinement/Decision_Log/Ephemeral)
   - Generate tags based on action type (e.g., "refactor", "bugfix", "feature", "test")
   - Generate tags based on code changes (e.g., "authentication", "database", "UI")
   - Provide a brief summary of changes
3. Apply generated changes to codebase
4. Capture after snapshot
5. Return result with code changes and metadata

**LLM Classification Prompt Template**:
```
Generate code for the following request and provide metadata:

User Prompt: {prompt}
Current Context: {context}

Please respond with:
1. Code changes
2. Classification: [Actionable|Refinement|Decision_Log|Ephemeral]
3. Action tags: (e.g., refactor, bugfix, feature, test, documentation)
4. Domain tags: (e.g., authentication, database, UI, API, performance)
5. Brief summary: What changed and why

Classification Guidelines:
- Actionable: Creates new functionality or makes substantive changes
- Refinement: Improves/modifies recent changes to the same files
- Decision_Log: Important architectural decision without code changes
- Ephemeral: Questions, clarifications, or off-topic discussions

Tag Guidelines:
Action Tags (what type of work):
- feature: New functionality
- bugfix: Fixing a defect
- refactor: Code restructuring without behavior change
- test: Adding or modifying tests
- documentation: Documentation updates
- performance: Performance optimization
- security: Security improvements

Domain Tags (what area of codebase):
- authentication: User auth/login
- database: Data persistence
- UI: User interface
- API: API endpoints/integration
- validation: Input validation
- error-handling: Error management
- configuration: Config changes
```

**Dependencies**: LLM code generation service (external)

### 2. Diff Analyzer

**Purpose**: Compute code changes resulting from a prompt and verify real impact.

**Interface**:
```
compute_diff(before_state: FileSystemSnapshot, after_state: FileSystemSnapshot) -> Diff

Diff = {
  files_added: List<FilePath>,
  files_deleted: List<FilePath>,
  files_modified: List<(FilePath, LineChanges)>,
  is_empty: Boolean
}

get_affected_files(diff: Diff) -> List<FilePath>
```

**Algorithm**:
1. Compare file system snapshots before and after prompt execution
2. Identify added, deleted, and modified files
3. For modified files, compute line-level changes
4. Mark diff as empty if no substantive changes exist

**Implementation Notes**:
- Use standard diff algorithms (e.g., Myers diff)
- Filter out whitespace-only changes
- Exclude configured file patterns (e.g., build artifacts)

### 3. Prompt Classifier with Verification

**Purpose**: Verify LLM-provided classification against actual diff and allow user overrides.

**Interface**:
```
verify_classification(
  llm_classification: PromptType,
  diff: Diff,
  user_override: Option<PromptType>
) -> PromptType

PromptType = Actionable | Refinement | Decision_Log | Ephemeral
```

**Algorithm**:
1. If user_override is provided, use it (user has final say)
2. If diff is empty but LLM classified as Actionable/Refinement:
   - Override to Decision_Log if user marked as important
   - Override to Ephemeral otherwise
   - Log warning about LLM misclassification
3. If diff is non-empty but LLM classified as Ephemeral/Decision_Log:
   - Override to Actionable
   - Log warning about LLM misclassification
4. Otherwise, trust LLM classification
5. Return final classification

**Verification Rules**:
- Non-empty diff MUST be Actionable or Refinement
- Empty diff MUST be Decision_Log or Ephemeral
- User override always takes precedence
- Log mismatches for LLM improvement

**Dependencies**: Diff Analyzer, Refinement Detector

### 4. Refinement Detector

**Purpose**: Identify when a prompt refines earlier work and build the prompt tree.

**Interface**:
```
detect_refinement(
  current_prompt: Prompt, 
  recent_prompts: List<Prompt>,
  prompt_tree: PromptDAG
) -> Option<PromptID>

Returns: Some(parent_prompt_id) if refinement detected, None otherwise
```

**Algorithm**:
1. Get files affected by current prompt
2. For each recent prompt (within time window):
   - Compute file overlap
   - Check prompt tree for potential parent relationships
   - If overlap > threshold AND time_delta < max_refinement_window:
     - Return parent prompt ID
3. Return None if no refinement detected

**Prompt Tree (DAG)**:
- Nodes: Prompts
- Edges: Refinement relationships, dependencies, context reuse
- Enables: Tracking evolution, preventing contradictions, reusing reasoning

**Configuration**:
- `refinement_time_window`: 30 minutes (default)
- `file_overlap_threshold`: 50% (default)

### 4. Refinement Detector

**Purpose**: Identify when a prompt refines earlier work and build the prompt tree.

**Interface**:
```
detect_refinement(
  current_prompt: Prompt, 
  recent_prompts: List<Prompt>,
  prompt_tree: PromptDAG
) -> Option<PromptID>

Returns: Some(parent_prompt_id) if refinement detected, None otherwise
```

**Algorithm**:
1. Get files affected by current prompt
2. For each recent prompt (within time window):
   - Compute file overlap
   - Check prompt tree for potential parent relationships
   - If overlap > threshold AND time_delta < max_refinement_window:
     - Return parent prompt ID
3. Return None if no refinement detected

**Prompt Tree (DAG)**:
- Nodes: Prompts
- Edges: Refinement relationships, dependencies, context reuse
- Enables: Tracking evolution, preventing contradictions, reusing reasoning

**Configuration**:
- `refinement_time_window`: 30 minutes (default)
- `file_overlap_threshold`: 50% (default)

### 5. Hierarchical Storage Manager

**Purpose**: Store prompts in date-based hierarchy and manage embeddings for search.

**Interface**:
```
store_prompt(prompt: PromptRecord, date: Date) -> Result<UUID>
get_prompts_by_date(date: Date) -> List<PromptRecord>
store_embedding(prompt_id: UUID, embedding: Vector) -> Result<()>
```

**Storage Structure**: Year/Month/Day/prompts.jsonl

**Algorithm**:
1. Determine storage path from prompt timestamp
2. Append prompt record to appropriate date file
3. Store embedding in vector DB for search
4. Update DAG with refinement edges

### 6. Prompt Tree Builder

**Purpose**: Construct and maintain DAG of prompt relationships.

**Interface**:
```
add_prompt_node(prompt_id: UUID) -> Result<()>
add_edge(from: UUID, to: UUID, edge_type: EdgeType) -> Result<()>
get_children(prompt_id: UUID) -> List<UUID>
get_parents(prompt_id: UUID) -> List<UUID>
```

**EdgeType**: Refinement | Dependency | ContextReuse

### 7. Task Grouper (On-Demand for Search/Reordering)

**Purpose**: Group related prompts into logical tasks using semantic similarity when user searches or reorders history.

**Interface**:
```
group_prompt_on_search(
  prompt: Prompt, 
  prompt_summary: String,
  vector_db: VectorDatabase
) -> TaskID

compute_embedding(text: String) -> Vector
search_similar(embedding: Vector, threshold: Float) -> List<(TaskID, Float)>
reorder_by_similarity(query: String) -> List<Task>
```

**Algorithm** (On-Demand Clustering):
1. When user searches or requests task-based view:
   - Generate embedding from search query or prompt summary
   - Search vector database for similar task embeddings
   - If similarity > threshold AND temporal_proximity:
     - Group prompts into tasks
   - Otherwise:
     - Create new task
2. Default chronological view does NOT use task grouping
3. Task grouping is computed dynamically based on user interaction

**Vector Database**:
- Stores prompt embeddings for fast similarity search
- Enables semantic grouping when requested
- Supports efficient nearest-neighbor queries
- NOT used for default chronological view

**Similarity Factors**:
- Semantic similarity of prompt text (primary, via embeddings)
- File overlap (secondary)
- Temporal proximity (tertiary)

**Configuration**:
- `similarity_threshold`: 0.7 (default)
- `max_task_time_span`: 2 hours (default)
- `embedding_model`: sentence-transformers or similar

**Usage Pattern**:
- Default: Date-based hierarchical view (no task grouping)
- On search: Use vector DB to find similar prompts and group into tasks
- On reorder: Use vector DB to reorganize history by semantic similarity

### 8. Summary Generator

**Purpose**: Create human-readable task summaries that merge refinements and capture the development narrative.

**Interface**:
```
generate_summary(
  task: Task, 
  prompts: List<Prompt>,
  prompt_tree: PromptDAG
) -> TaskSummary

TaskSummary = {
  title: String,
  description: String,
  files_changed: List<FilePath>,
  key_decisions: List<String>,
  final_diff: Diff,
  tags: List<String>
}
```

**Algorithm**:
1. Extract intent from first actionable prompt
2. Trace refinement chain through prompt tree
3. Identify final outcome from last prompt
4. Merge intermediate refinements into narrative (don't list each iteration)
5. Extract key decisions from Decision_Log prompts in task
6. List affected files and compute cumulative diff
7. Generate tags based on prompt content and changes

**Template**:
```
Title: [Extracted from first prompt]

Description:
[Original intent] → [Final outcome]
[Key decisions made during refinements]

Files Changed:
- file1.py
- file2.ts

Tags: [critical, refactor, validation, etc.]

Diff: [Cumulative changes]
```

**AI-Generated Summaries**: Use LLM to generate concise, human-readable summaries from prompt text and diffs

### 9. Tag Manager

**Purpose**: Manage and normalize tags generated by the LLM to ensure consistency.

**Interface**:
```
normalize_tags(llm_tags: List<String>) -> NormalizedTags

NormalizedTags = {
  action_tags: List<ActionTag>,
  domain_tags: List<DomainTag>
}

ActionTag = Feature | Bugfix | Refactor | Test | Documentation | Performance | Security
DomainTag = String  // Free-form domain tags (e.g., "authentication", "database")
```

**Algorithm**:
1. Parse LLM-provided tags
2. Categorize into action tags and domain tags
3. Normalize action tags to predefined set
4. Keep domain tags as-is for flexibility
5. Remove duplicates
6. Return normalized tags

**Tag Categories**:

**Action Tags** (predefined, normalized):
- `feature`: New functionality added
- `bugfix`: Defect or error fixed
- `refactor`: Code restructuring without behavior change
- `test`: Test creation or modification
- `documentation`: Documentation updates
- `performance`: Performance optimization
- `security`: Security improvements

**Domain Tags** (free-form, LLM-generated):
- Examples: `authentication`, `database`, `UI`, `API`, `validation`, `error-handling`
- Not restricted to predefined set
- Allows flexibility for project-specific domains
- Can be used for filtering and search

**Tag Usage**:
- Tags enable filtering prompts by type of work or domain
- Tags improve search relevance in vector DB
- Tags help generate better task summaries
- Tags provide quick visual indicators in UI

### 6. Storage Manager

**Purpose**: Persist prompts, tasks, metadata, and embeddings to disk and vector database.

**Data Model**:
```
PromptRecord = {
  id: UUID,
  text: String,
  classification: PromptType,
  timestamp: Timestamp,
  diff: Diff,
  task_id: UUID,
  parent_prompt_id: Option<UUID>,
  tags: List<String>,
  summary: String
}

TaskRecord = {
  id: UUID,
  prompt_ids: List<UUID>,
  summary: TaskSummary,
  created_at: Timestamp,
  compressed: Boolean,
  embedding: Vector
}

PromptDAG = {
  nodes: Map<UUID, PromptNode>,
  edges: List<(UUID, UUID, EdgeType)>
}

EdgeType = Refinement | Dependency | ContextReuse

Configuration = {
  similarity_threshold: Float,
  gc_schedule: CronExpression,
  gc_age_threshold: Duration,
  excluded_patterns: List<GlobPattern>,
  storage_path: FilePath,
  vector_db_config: VectorDBConfig
}
```

**Interface**:
```
store_prompt(prompt: PromptRecord) -> Result<UUID>
store_task(task: TaskRecord) -> Result<UUID>
store_dag_edge(from: UUID, to: UUID, edge_type: EdgeType) -> Result<()>
get_prompt(id: UUID) -> Option<PromptRecord>
get_task(id: UUID) -> Option<TaskRecord>
get_prompt_tree() -> PromptDAG
query_prompts(filter: QueryFilter) -> List<PromptRecord>
query_tasks(filter: QueryFilter) -> List<TaskRecord>
```

**Storage Format**: 
- Structured data: JSON Lines (JSONL) for append-only efficiency
- Vector embeddings: Vector database (e.g., Qdrant, Weaviate, or embedded solution)

**File Structure**:
```
.prompt-history/
  2024/                   # Year
    01/                   # Month
      15/                 # Day
        prompts.jsonl     # Prompts for this day
    02/
      10/
        prompts.jsonl
  dag.jsonl               # Prompt tree edges (global)
  config.json             # Configuration
  vector_db/              # Vector database files (for semantic search)
    embeddings.db
  index/                  # Search indices
    keywords.idx          # Inverted index: keyword -> [prompt_ids]
    tags.idx              # Tag index: tag -> [prompt_ids]
    by_file.idx           # File index: filepath -> [prompt_ids]
    by_task.idx           # Task groupings (computed on-demand)
```

**Storage Strategy**:
- **Default**: Date-based hierarchical storage (Year/Month/Day)
- **Keyword Search**: Inverted index for fast keyword lookup
- **Tag Search**: Tag index for filtering by action/domain tags
- **Semantic Search**: Vector DB for conceptual similarity
- **Chronological access**: Fast (direct file path from date)
- **Combined Search**: Multiple indices can be queried together

### 7. Garbage Collector

**Purpose**: Compress old prompt history to manage storage.

**Interface**:
```
run_gc(age_threshold: Duration) -> GCResult

GCResult = {
  tasks_compressed: Int,
  prompts_removed: Int,
  storage_freed: Bytes
}
```

**Algorithm**:
1. Identify tasks older than age_threshold
2. For each old task:
   - Keep task summary
   - Remove individual prompt details
   - Mark task as compressed
3. Never delete Decision_Log prompts
4. Update indices

**Configuration**:
- `gc_age_threshold`: 90 days (default)
- `gc_schedule`: Weekly (default)

### 8. Query Interface with Keyword Search

**Purpose**: Enable flexible retrieval of prompt history using multiple search methods.

**Interface**:
```
QueryFilter = {
  date_range: Option<(Timestamp, Timestamp)>,
  file_paths: Option<List<FilePath>>,
  task_ids: Option<List<UUID>>,
  classifications: Option<List<PromptType>>,
  tags: Option<List<String>>,
  keywords: Option<List<String>>,
  semantic_query: Option<String>
}

query(filter: QueryFilter) -> QueryResult

QueryResult = {
  tasks: List<TaskRecord>,
  prompts: List<PromptRecord>,
  relevance_scores: Map<UUID, Float>  // For semantic/keyword search
}

// Specialized search methods
search_by_keywords(keywords: List<String>, options: SearchOptions) -> QueryResult
search_by_tags(tags: List<String>, match_mode: TagMatchMode) -> QueryResult
search_semantic(query: String, threshold: Float) -> QueryResult
```

**Search Types**:

1. **Keyword Search**: Full-text search across prompt text and summaries
   - Supports multiple keywords (AND/OR logic)
   - Case-insensitive matching
   - Searches in: prompt text, summary, file paths
   - Returns results ranked by relevance

2. **Tag-Based Search**: Filter by action tags or domain tags
   - Match modes: ANY (OR), ALL (AND), EXACT
   - Example: Find all "bugfix" prompts in "authentication" domain

3. **Semantic Search**: Vector similarity search
   - Uses embeddings to find conceptually similar prompts
   - Example: "login issues" finds prompts about authentication, sessions, etc.

4. **Combined Search**: Mix multiple filter types
   - Example: Keywords + date range + tags

**SearchOptions**:
```
SearchOptions = {
  match_mode: MatchMode,        // AND, OR, EXACT
  case_sensitive: Boolean,
  include_summaries: Boolean,   // Search in summaries too
  include_file_paths: Boolean,  // Search in affected file paths
  max_results: Int,
  sort_by: SortOrder            // Relevance, Date, FileCount
}

MatchMode = AND | OR | EXACT
TagMatchMode = ANY | ALL | EXACT
SortOrder = Relevance | DateAsc | DateDesc | FileCount
```

**Algorithm** (Keyword Search):
1. Parse keywords from user input
2. Build search index from:
   - Prompt text
   - Summaries
   - File paths (if enabled)
   - Tags
3. For each prompt in storage:
   - Compute keyword match score
   - Apply filters (date, classification, tags)
4. Rank results by relevance score
5. Return top N results

**Search Index Structure**:
```
.prompt-history/
  index/
    keywords.idx          # Inverted index: keyword -> [prompt_ids]
    tags.idx              # Tag index: tag -> [prompt_ids]
    files.idx             # File index: filepath -> [prompt_ids]
```

**Implementation Notes**:
- Use inverted index for fast keyword lookup
- Cache frequently searched keywords
- Support fuzzy matching for typos
- Highlight matching keywords in results

**Example Queries**:

```python
# Search by keywords
query(QueryFilter(keywords=["authentication", "login"]))

# Search by tags
query(QueryFilter(tags=["bugfix", "security"]))

# Combined search
query(QueryFilter(
  keywords=["database"],
  tags=["performance"],
  date_range=(start_date, end_date)
))

# Semantic search
search_semantic("how did we handle user sessions?", threshold=0.7)
```

### 9. Export Engine

**Purpose**: Generate human-readable documentation from prompt history for display in editor and export.

**Interface**:
```
export_markdown(filter: QueryFilter) -> String
export_json(filter: QueryFilter) -> JSON
generate_editor_view(task_id: UUID) -> EditorView

EditorView = {
  task_summary: TaskSummary,
  ordered_prompts: List<PromptSummary>,
  key_decisions: List<String>,
  code_impact: Diff
}
```

**Markdown Format**:
```markdown
# Development History

## Task: [Task Title]
**Date**: [Timestamp]
**Files**: file1.py, file2.ts
**Tags**: critical, refactor

[Task summary description]

### Key Decisions
- [Decision 1]
- [Decision 2]

### Changes
```diff
[Cumulative diff]
```

---

[Next task...]
```

**Editor Integration**:
- Display task-wise documentation alongside code
- Show prompt tree visualization
- Enable navigation through refinement history
- Support inline viewing of decision logs

## Data Models

### Core Types

```
PromptType = Actionable | Refinement | Decision_Log | Ephemeral

Prompt = {
  id: UUID,
  text: String,
  timestamp: Timestamp,
  classification: PromptType,
  diff: Diff,
  task_id: UUID,
  parent_id: Option<UUID>,
  tags: List<String>,
  summary: String
}

Diff = {
  files_added: List<FilePath>,
  files_deleted: List<FilePath>,
  files_modified: List<(FilePath, LineChanges)>,
  is_empty: Boolean
}

LineChanges = {
  additions: List<(LineNumber, String)>,
  deletions: List<(LineNumber, String)>
}

Task = {
  id: UUID,
  prompt_ids: List<UUID>,
  summary: TaskSummary,
  created_at: Timestamp,
  files_affected: Set<FilePath>,
  compressed: Boolean,
  embedding: Vector
}

TaskSummary = {
  title: String,
  description: String,
  files_changed: List<FilePath>,
  key_decisions: List<String>,
  final_diff: Diff,
  tags: List<String>
}

PromptDAG = {
  nodes: Map<UUID, PromptNode>,
  edges: List<Edge>
}

PromptNode = {
  prompt_id: UUID,
  children: List<UUID>,
  parents: List<UUID>
}

Edge = {
  from: UUID,
  to: UUID,
  edge_type: EdgeType
}

EdgeType = Refinement | Dependency | ContextReuse
```

### Relationships

- Each Prompt belongs to exactly one Task
- Each Task contains one or more Prompts
- Refinement Prompts reference a parent Actionable Prompt via parent_id
- Tasks are independent (no parent-child relationships)
- The PromptDAG captures refinement chains, dependencies, and context reuse
- DAG edges enable:
  - **Refinement**: Prompt B refines Prompt A
  - **Dependency**: Prompt B depends on context from Prompt A
  - **ContextReuse**: Prompt B reuses reasoning from Prompt A

### Vector Embeddings

- Each Task has an associated embedding vector
- Embeddings enable semantic similarity search
- Stored in vector database for efficient nearest-neighbor queries
- Updated when new prompts are added to tasks


## Correctness Properties

*A property is a characteristic or behavior that should hold true across all valid executions of a system—essentially, a formal statement about what the system should do. Properties serve as the bridge between human-readable specifications and machine-verifiable correctness guarantees.*

### Classification Properties

**Property 1: Non-empty diff implies Actionable or Refinement**
*For any* prompt with a non-empty code diff, the classifier should mark it as either Actionable or Refinement (never Decision_Log or Ephemeral).
**Validates: Requirements 1.2**

**Property 2: Empty diff with importance flag implies Decision_Log**
*For any* prompt with an empty code diff and user_marked_important=true, the classifier should mark it as Decision_Log.
**Validates: Requirements 1.3**

**Property 3: Ephemeral prompts are not persisted**
*For any* prompt classified as Ephemeral, querying the storage after persistence should not return that prompt.
**Validates: Requirements 1.5**

### Diff Analysis Properties

**Property 4: Diff captures all change types**
*For any* file system state transition, the computed diff should include all file additions, deletions, and modifications that occurred.
**Validates: Requirements 2.2**

**Property 5: Empty diff triggers reclassification**
*For any* prompt that produces an empty diff, the system should reclassify it as either Decision_Log or Ephemeral (never Actionable or Refinement).
**Validates: Requirements 2.3**

**Property 6: Prompt-diff association is maintained**
*For any* prompt with a computed diff, retrieving that prompt from storage should return the same diff that was originally computed.
**Validates: Requirements 2.4**

### Refinement Detection Properties

**Property 7: File overlap and temporal proximity detect refinements**
*For any* prompt that modifies files changed by a recent prompt (within the time window) with sufficient file overlap, the refinement detector should classify it as a Refinement and link it to the parent prompt.
**Validates: Requirements 3.1, 3.2**

**Property 8: Refinement parent links are preserved**
*For any* prompt classified as a Refinement, retrieving it from storage should return a valid parent_id that references an Actionable prompt.
**Validates: Requirements 3.3**

**Property 9: Multiple refinements per actionable prompt**
*For any* Actionable prompt, the system should support storing and retrieving multiple Refinement prompts that reference it as their parent.
**Validates: Requirements 3.4**

### Task Grouping Properties

**Property 10: Similarity above threshold groups prompts**
*For any* two prompts with semantic similarity above the configured threshold and within temporal proximity, the task grouper should assign them to the same task.
**Validates: Requirements 4.2, 4.3**

**Property 11: Task IDs are unique**
*For any* set of tasks created by the system, all task IDs should be distinct.
**Validates: Requirements 4.4**

**Property 12: Each prompt belongs to exactly one task**
*For any* prompt in the system, it should be associated with exactly one task ID (not zero, not multiple).
**Validates: Requirements 4.5**

### Summary Generation Properties

**Property 13: Multi-prompt tasks produce single summary**
*For any* task containing multiple prompts, the summary generator should produce exactly one TaskSummary.
**Validates: Requirements 5.1**

**Property 14: Summary contains required elements**
*For any* generated task summary, it should include the original intent, final outcome, affected files, and key decisions.
**Validates: Requirements 5.2, 5.4**

**Property 15: Refinements are merged in summary**
*For any* task with refinement prompts, the generated summary should not list individual refinement iterations separately.
**Validates: Requirements 5.3**

### Storage Properties

**Property 16: Non-ephemeral prompts are persisted**
*For any* prompt classified as Actionable, Refinement, or Decision_Log, it should be retrievable from storage after persistence.
**Validates: Requirements 6.1**

**Property 17: Stored records contain required fields**
*For any* prompt or task stored in the system, retrieving it should return all required fields (prompt text, classification, timestamp, diff for prompts; task ID, prompt IDs, summary for tasks).
**Validates: Requirements 6.2, 6.3**

**Property 18: Query filtering works for all filter types**
*For any* query filter (date range, file path, task ID, or classification type), the query interface should return only prompts/tasks matching that filter.
**Validates: Requirements 8.1, 8.2, 8.3, 8.4**

**Property 19: Keyword search returns matching prompts**
*For any* keyword search query, the query interface should return only prompts whose text or summary contains the specified keywords.
**Validates: Requirements 8.5**

**Property 20: Tag search filters correctly**
*For any* tag-based search, the query interface should return only prompts that have all specified tags (AND mode) or any specified tags (OR mode).
**Validates: Requirements 8.6**

**Property 21: Semantic search returns similar prompts**
*For any* semantic search query, the query interface should return prompts with embedding similarity above the threshold, ranked by relevance.
**Validates: Requirements 8.7**

**Property 22: Combined search applies all filters**
*For any* query with multiple filter types (keywords + tags + date range), the query interface should return only prompts matching ALL specified criteria.
**Validates: Requirements 8.8**

**Property 23: Query results include summaries and diffs**
*For any* query result, each returned task should include its summary and each returned prompt should include its associated diff.
**Validates: Requirements 8.9**

**Property 24: Search results are ranked by relevance**
*For any* keyword or semantic search, results should be ordered by relevance score with highest scores first.
**Validates: Requirements 8.10**

### Garbage Collection Properties

**Property 25: Old tasks are compressed by threshold**
*For any* task older than the configured age threshold, running garbage collection should compress it (preserve summary, remove prompt details).
**Validates: Requirements 7.2**

**Property 26: GC preserves summaries and discards details**
*For any* task compressed by garbage collection, the task summary should remain retrievable while individual prompt details should be removed.
**Validates: Requirements 7.3**

**Property 27: Decision_Log prompts are never deleted**
*For any* prompt classified as Decision_Log, running garbage collection (regardless of age) should never delete it.
**Validates: Requirements 7.4**

**Property 28: GC maintains referential integrity**
*For any* task relationships before garbage collection, all valid task references should remain valid after compression.
**Validates: Requirements 7.5**

### Export Properties

**Property 29: Markdown export organizes by tasks**
*For any* set of tasks exported to Markdown, the output should contain one section per task with its summary.
**Validates: Requirements 9.3**

**Property 30: Exports include diffs in readable format**
*For any* prompt with a diff exported to Markdown or JSON, the output should include the diff in a formatted, readable representation.
**Validates: Requirements 9.4**

**Property 31: Filtered export matches query**
*For any* export with a query filter, the exported history should contain only tasks and prompts matching that filter.
**Validates: Requirements 9.5**

### Configuration Properties

**Property 32: Exclusion patterns filter diffs**
*For any* configured file exclusion pattern, files matching that pattern should not appear in computed diffs.
**Validates: Requirements 10.3**

**Property 33: Configuration changes apply immediately**
*For any* configuration parameter change, the new value should be used by the system without requiring a restart.
**Validates: Requirements 10.5**

### Tag Properties

**Property 34: LLM generates action and domain tags**
*For any* prompt that generates code, the LLM should provide both action tags and domain tags.
**Validates: Requirements 11.1, 11.2**

**Property 35: Action tags are normalized**
*For any* LLM-generated action tag, the Tag Manager should normalize it to the predefined set of action tags.
**Validates: Requirements 11.3**

**Property 36: Domain tags are preserved**
*For any* LLM-generated domain tag, the Tag Manager should preserve it as-is without normalization.
**Validates: Requirements 11.4**

**Property 37: Tags enable search and filtering**
*For any* tag-based query, the system should return prompts that match the specified tags.
**Validates: Requirements 11.5**

## Error Handling

The system should handle errors gracefully and provide clear feedback:

### Classification Errors
- **Invalid prompt input**: Return error with message "Prompt text cannot be empty"
- **Diff computation failure**: Log error, classify as Ephemeral, continue processing
- **Semantic similarity computation failure**: Log error, create new task for prompt

### Storage Errors
- **Disk full**: Return error "Storage full, cannot persist prompt history"
- **Corrupted storage file**: Attempt recovery from backup, log error
- **Invalid query filter**: Return error "Invalid filter parameters: [details]"

### Garbage Collection Errors
- **Compression failure**: Log error, skip task, continue with next
- **Referential integrity violation**: Abort GC, log error, alert user

### Export Errors
- **Invalid format request**: Return error "Unsupported export format: [format]"
- **Empty result set**: Return empty document with message "No history matches filter"

### Configuration Errors
- **Invalid threshold value**: Return error "Threshold must be between 0.0 and 1.0"
- **Invalid file pattern**: Return error "Invalid glob pattern: [pattern]"
- **Missing storage path**: Create default path, log warning

## Testing Strategy

The system will be tested using a dual approach combining unit tests and property-based tests:

### Unit Testing
Unit tests will focus on:
- Specific examples of prompt classification (e.g., "How do I...?" → Ephemeral)
- Edge cases (empty prompts, very large diffs, boundary threshold values)
- Error conditions (storage failures, invalid configurations)
- Integration points between components

### Property-Based Testing
Property-based tests will verify universal properties across all inputs:
- Each correctness property listed above will be implemented as a property-based test
- Tests will use a property-based testing library appropriate for the implementation language
- Each test will run a minimum of 100 iterations with randomized inputs
- Each test will be tagged with: **Feature: prompt-history-manager, Property N: [property text]**

### Test Data Generation
For property-based tests, we will generate:
- Random prompts with varying content and characteristics
- Random file system states and diffs
- Random task groupings with varying similarity scores
- Random timestamps within configurable ranges
- Random configurations within valid bounds

### Testing Libraries
- **Python**: Hypothesis
- **TypeScript/JavaScript**: fast-check
- **Java**: jqwik
- **Rust**: proptest

### Coverage Goals
- Unit test coverage: 80% of lines
- Property test coverage: All 28 correctness properties
- Integration test coverage: All component interactions
