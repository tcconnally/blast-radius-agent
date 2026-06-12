---
name: blast-radius
description: Analyze the blast radius of code changes using GitLab Orbit's knowledge graph. Trace cross-file dependencies and assess impact before making changes.
---

# Blast Radius Analysis

Use GitLab Orbit's knowledge graph to find all code that depends on a given file, function, or class. This helps developers understand what will break before they make changes.

## When to Use

- A developer wants to know the impact of changing a specific file or function
- Someone mentions `@blast-radius <file-path>` in an issue or MR comment
- You're asked to assess risk before a refactor or migration

## Query Pattern

### Step 1: Understand the Schema

Use `get_graph_schema` to list available node types and relationships in the Orbit knowledge graph. Key types:
- `gl_definition` — files, functions, classes, modules
- `gl_reference` — edges connecting definitions (source depends on target)

### Step 2: Find the Definition

Query `gl_definition` nodes matching the target file or function:

```sql
-- Orbit SQL (CLI local mode)
SELECT * FROM gl_definition WHERE path LIKE '%target_file.py%' OR name = 'functionName'
```

Or via MCP: use `query_graph` with a filter on name or path.

### Step 3: Trace Direct References

Find all `gl_reference` edges pointing TO the definition. These are the direct dependents:

```sql
-- What depends on target?
SELECT DISTINCT t2.name, t2.path, t2.language
FROM gl_definition t1
JOIN gl_reference ON t1.id = gl_reference.target_id
JOIN gl_definition t2 ON gl_reference.source_id = t2.id
WHERE t1.path LIKE '%target_file.py%'
```

### Step 4: Trace Transitive Impact

For each direct dependent, repeat Step 3 to find files that depend on THEM:

```sql
-- Full transitive chain (2 levels)
WITH direct AS (
  SELECT t2.id, t2.name, t2.path
  FROM gl_definition t1
  JOIN gl_reference ON t1.id = gl_reference.target_id
  JOIN gl_definition t2 ON gl_reference.source_id = t2.id
  WHERE t1.path LIKE '%target_file.py%'
)
SELECT DISTINCT t3.name, t3.path, t3.language
FROM direct d
JOIN gl_reference ON d.id = gl_reference.target_id
JOIN gl_definition t3 ON gl_reference.source_id = t3.id
WHERE t3.id NOT IN (SELECT id FROM direct)
```

### Step 5: Assess Cross-Project Impact

If the knowledge graph spans multiple projects, identify which projects contain dependents:

```sql
SELECT DISTINCT t2.project_path
FROM gl_definition t1
JOIN gl_reference ON t1.id = gl_reference.target_id
JOIN gl_definition t2 ON gl_reference.source_id = t2.id
WHERE t1.path LIKE '%target_file.py%'
```

## Risk Classification

| Risk Level | Criteria |
|---|---|
| **Low** | 0-2 direct dependents, no transitive dependents |
| **Medium** | 3-10 direct dependents, or 1-5 transitive |
| **High** | 10-50 direct dependents, or 5-20 transitive |
| **Critical** | 50+ dependents, or hits a shared library used across 3+ projects |

## Response Format

Always structure your findings as:

```
DIRECT IMPACT:
- <file>:<line> — <how it references the target>
...

TRANSITIVE IMPACT:
- <file> — depends on <direct-dependent>
...

PROJECT IMPACT:
- <project-path>

RISK SCORE: <Low|Medium|High|Critical>
RECOMMENDATION: <suggested action before landing the change>
```

## Orbit CLI Quick Reference

For local mode (Orbit CLI):

```bash
# Index a project
orbit index /path/to/project

# Basic query — all definitions
orbit sql "SELECT * FROM gl_definition LIMIT 20"

# Dependencies of a file
orbit sql "SELECT t2.path, t2.name FROM gl_definition t1 JOIN gl_reference ON t1.id = gl_reference.target_id JOIN gl_definition t2 ON gl_reference.source_id = t2.id WHERE t1.path LIKE '%target%'"

# List relationship types
orbit sql "SELECT DISTINCT relationship_type FROM gl_reference"

# Count dependents per file (hotspot analysis)
orbit sql "SELECT t1.path, COUNT(gl_reference.source_id) as dependents FROM gl_definition t1 JOIN gl_reference ON t1.id = gl_reference.target_id GROUP BY t1.path ORDER BY dependents DESC LIMIT 20"
```

## Pitfalls

- **Cyclic dependencies**: If the graph has cycles, transitive traversal may loop. Limit traversal depth to 3 levels.
- **Dynamic imports**: `import()` and `require()` may not appear in static analysis. Flag these as "unanalyzed."
- **Test files**: Dependencies to/from test files may inflate blast radius. Consider filtering `*test*` and `*spec*` paths.
- **Generated code**: Auto-generated files create false positives. Filter `*.generated.*` and `__generated__/` paths.
