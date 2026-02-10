# Design Updates: LLM-Integrated Classification & Keyword Search

## Summary

Updated the Prompt History Manager design to reflect that the code-generating LLM simultaneously handles classification and tagging, rather than having a separate post-hoc classification step. Also added comprehensive keyword-based search capabilities.

## Key Changes

### 1. **LLM-Integrated Classification** (Step 2 in pipeline)
- The LLM that generates code now also:
  - Classifies the prompt type (Actionable/Refinement/Decision_Log/Ephemeral)
  - Generates action tags (feature, bugfix, refactor, test, etc.)
  - Generates domain tags (authentication, database, UI, etc.)
  - Provides a brief summary of changes

### 2. **Verification Step** (Step 3 in pipeline)
- Added verification logic to check LLM classification against actual diff
- Ensures consistency (non-empty diff must be Actionable/Refinement)
- Logs mismatches for LLM improvement
- User override always takes precedence

### 3. **Tag Management**
- Added Tag Manager component to normalize and categorize tags
- Two tag types:
  - **Action Tags**: Predefined set (feature, bugfix, refactor, test, documentation, performance, security)
  - **Domain Tags**: Free-form, project-specific (authentication, database, UI, API, etc.)

### 4. **LLM Prompt Template**
- Added detailed prompt template for LLM to follow
- Includes classification guidelines
- Includes tag generation guidelines with examples

### 5. **Keyword Search Capabilities** (NEW)
- Added comprehensive keyword search across prompt text and summaries
- Multiple search types:
  - **Keyword Search**: Full-text search with AND/OR logic
  - **Tag-Based Search**: Filter by action tags or domain tags
  - **Semantic Search**: Vector similarity for conceptual queries
  - **Combined Search**: Mix keywords + tags + date range + file paths
- Search indices:
  - Inverted index for keywords
  - Tag index for fast tag filtering
  - File index for file-based queries
- Relevance ranking for search results
- Support for fuzzy matching and case-insensitive search

## Benefits

1. **Efficiency**: LLM already understands intent and impact, so classification happens naturally
2. **Accuracy**: LLM has full context while generating code, leading to better classification
3. **Rich Metadata**: Tags provide additional context beyond simple classification
4. **Verification**: Diff-based verification ensures LLM classification is accurate
5. **Flexibility**: Domain tags are free-form, allowing project-specific categorization
6. **Powerful Search**: Multiple search methods (keyword, tag, semantic) enable quick discovery
7. **Discoverability**: Users can find prompts by keywords, tags, files, dates, or natural language

## Architecture Impact

- **Before**: Generate code → Compute diff → Classify based on diff
- **After**: Generate code + classify + tag → Compute diff → Verify classification

The verification step ensures the LLM's classification matches reality, with user override as the final authority.

## Search Architecture

- **Default View**: Chronological (date-based hierarchy)
- **Keyword Search**: Inverted index for fast lookup
- **Tag Search**: Dedicated tag index
- **Semantic Search**: Vector DB for conceptual similarity
- **Combined**: All indices can be queried together with relevance ranking
