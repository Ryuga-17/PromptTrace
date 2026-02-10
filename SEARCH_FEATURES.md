# Keyword Search Features

## Overview

The Prompt History Manager now supports comprehensive keyword-based search capabilities, allowing users to find prompts using multiple search methods.

## Search Types

### 1. Keyword Search
Full-text search across prompt text and summaries.

**Features**:
- Multiple keywords with AND/OR logic
- Case-insensitive matching
- Searches in: prompt text, summary, file paths
- Results ranked by relevance
- Fuzzy matching for typos

**Example**:
```python
# Find prompts containing "authentication" AND "login"
query(QueryFilter(keywords=["authentication", "login"]))
```

### 2. Tag-Based Search
Filter by action tags or domain tags.

**Match Modes**:
- `ANY` (OR): Match any of the specified tags
- `ALL` (AND): Match all specified tags
- `EXACT`: Exact tag match

**Example**:
```python
# Find all bugfix prompts in authentication domain
query(QueryFilter(tags=["bugfix", "authentication"]))
```

### 3. Semantic Search
Vector similarity search using natural language queries.

**Features**:
- Conceptual similarity (not just keyword matching)
- Example: "login issues" finds prompts about authentication, sessions, tokens, etc.
- Configurable similarity threshold
- Results ranked by semantic relevance

**Example**:
```python
# Find prompts conceptually related to user sessions
search_semantic("how did we handle user sessions?", threshold=0.7)
```

### 4. Combined Search
Mix multiple filter types for precise queries.

**Example**:
```python
# Find database-related performance improvements from last month
query(QueryFilter(
  keywords=["database"],
  tags=["performance"],
  date_range=(start_date, end_date)
))
```

## Search Indices

The system maintains multiple indices for fast search:

1. **Keyword Index** (`keywords.idx`): Inverted index mapping keywords to prompt IDs
2. **Tag Index** (`tags.idx`): Maps tags to prompt IDs
3. **File Index** (`by_file.idx`): Maps file paths to prompt IDs
4. **Vector DB** (`embeddings.db`): Stores embeddings for semantic search

## Search Options

```python
SearchOptions = {
  match_mode: MatchMode,        # AND, OR, EXACT
  case_sensitive: Boolean,
  include_summaries: Boolean,   # Search in summaries too
  include_file_paths: Boolean,  # Search in affected file paths
  max_results: Int,
  sort_by: SortOrder            # Relevance, Date, FileCount
}
```

## Use Cases

### 1. Find All Authentication Changes
```python
query(QueryFilter(keywords=["authentication", "login", "auth"]))
```

### 2. Find Recent Bugfixes
```python
query(QueryFilter(
  tags=["bugfix"],
  date_range=(last_week, today)
))
```

### 3. Find Performance Optimizations in Database Code
```python
query(QueryFilter(
  tags=["performance", "database"],
  match_mode=ALL
))
```

### 4. Find Prompts That Modified Specific Files
```python
query(QueryFilter(file_paths=["src/auth/login.ts"]))
```

### 5. Natural Language Search
```python
search_semantic("how did we implement rate limiting?")
```

## Performance

- **Keyword Search**: O(log n) using inverted index
- **Tag Search**: O(1) using tag index
- **Semantic Search**: O(log n) using vector DB nearest-neighbor
- **Combined Search**: Intersection of multiple indices

## Future Enhancements

1. **Autocomplete**: Suggest keywords and tags as user types
2. **Search History**: Remember recent searches
3. **Saved Searches**: Save frequently used queries
4. **Advanced Filters**: Regular expressions, date ranges with natural language ("last week")
5. **Search Analytics**: Track which searches are most common
