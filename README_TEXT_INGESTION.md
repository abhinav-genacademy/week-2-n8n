# GraphRAG Ingestion from Natural Language Text

This workflow extracts structured knowledge graph data from unstructured natural language text using LLM-based entity extraction.

## Use Cases

Perfect for when you have:
- **Meeting notes** - Extract people, decisions, and action items
- **Project updates** - Identify projects, contributors, and status
- **Email threads** - Parse discussions and decisions
- **Reports** - Extract key findings and recommendations
- **Documentation** - Identify components, dependencies, and owners
- **Slack/Teams exports** - Build org knowledge from chat history

## How It Works

```
Text Document → LLM Extraction → Structured JSON → Neo4j Graph
```

1. **Fetch text** from GitHub (or local files)
2. **LLM extracts** entities (people, projects, decisions, documents)
3. **Parse** LLM response into structured JSON
4. **Load** into Neo4j with relationships
5. **Generate embeddings** for semantic search

## Workflow Comparison

| Workflow | Input Format | Best For |
|----------|-------------|----------|
| `n8n_ingestion_fixed_v2.json` | Structured JSON | Pre-structured data, APIs, databases |
| `n8n_ingestion_from_text.json` | Natural language text | Meeting notes, emails, reports, docs |

## Setup Instructions

### 1. Prepare Your Text Files

Upload `.txt` files to GitHub with natural language content:

```
meeting_notes_q3.txt
project_updates_sept.txt
engineering_retro.txt
```

### 2. Configure the Workflow

1. Import `n8n_ingestion_from_text.json` into n8n
2. Open **Document Manifest** node
3. Update the file list:

```javascript
const BASE = 'https://raw.githubusercontent.com/YOUR_USERNAME/YOUR_REPO/main/';

const docs = [
  { 
    url: BASE + 'meeting_notes_q3.txt',
    label: 'Q3 Engineering Meeting Notes',
    type: 'meeting_notes'
  },
  { 
    url: BASE + 'project_updates.txt',
    label: 'Project Status Updates',
    type: 'status_update'
  }
];
```

### 3. Configure Credentials

**Required credentials:**
- **Groq API** - For LLM entity extraction (free tier available)
- **Neo4j Basic Auth** - Your Neo4j credentials
- **HuggingFace API** - For embeddings (optional, if you add embedding step)

### 4. Run the Workflow

Execute the workflow. It will:
1. Fetch each text file
2. Send to LLM for entity extraction
3. Parse the structured response
4. Load into Neo4j

## What the LLM Extracts

The LLM is prompted to extract:

**Entities:**
- **People** - name, role, team
- **Projects** - name, description, status, tools
- **Decisions** - title, rationale, date, approver
- **Documents** - title, content, type, author

**Relationships:**
- Who worked on which projects
- Who approved which decisions
- Which documents relate to projects/decisions
- Which tools were used in projects

## Example Input Text

```text
Q3 2024 Engineering Meeting Notes
Date: September 15, 2024

Auth System Rebuild (Lead: Marcus Webb)
- Migration to OAuth 2.0 is 60% complete
- Using Auth0 and Vault
- Contributors: Marcus Webb, Jin Park

Decision: Adopt OAuth 2.0 with PKCE
- Approved by: Sarah Chen
- Date: August 15, 2024
- Rationale: Better mobile support, lower complexity
```

## Example LLM Output

```json
{
  "people": [
    {"id": "p001", "name": "Marcus Webb", "role": "Senior Engineer", "team": "Engineering"},
    {"id": "p002", "name": "Jin Park", "role": "Engineer", "team": "Engineering"},
    {"id": "p003", "name": "Sarah Chen", "role": "VP Engineering", "team": "Engineering"}
  ],
  "projects": [
    {
      "id": "pr001",
      "name": "Auth System Rebuild",
      "description": "Migration to OAuth 2.0",
      "status": "in_progress",
      "tools": ["Auth0", "Vault"],
      "contributors": ["p001", "p002"]
    }
  ],
  "decisions": [
    {
      "id": "d001",
      "title": "Adopt OAuth 2.0 with PKCE",
      "rationale": "Better mobile support, lower complexity",
      "date": "2024-08-15",
      "approved_by": "p003",
      "project_id": "pr001"
    }
  ]
}
```

## Tips for Better Extraction

### 1. Structure Your Text

Use clear section headers:
```
=== Project Updates ===
=== Key Decisions ===
=== Team Members ===
```

### 2. Include Key Information

For **people**: name, role, team
```
Sarah Chen (VP Engineering, Engineering team) leads...
```

For **projects**: name, status, tools, contributors
```
Auth System Rebuild (Lead: Marcus Webb)
Status: 60% complete
Tools: Auth0, Vault
Contributors: Marcus, Jin
```

For **decisions**: title, approver, date, rationale
```
Decision: Adopt OAuth 2.0
Approved by: Sarah Chen
Date: August 15, 2024
Rationale: Better mobile support...
```

### 3. Be Explicit About Relationships

```
Marcus Webb led the OAuth 2.0 migration
Jin Park contributed to the Auth System Rebuild
Sarah Chen approved the decision to adopt OAuth 2.0
```

## Customizing the Extraction Prompt

Edit the **Extract Entities** node's system message to:

1. **Add new entity types:**
```
- Teams: name, description
- Skills: name, category
- Tools: name, vendor, purpose
```

2. **Change extraction rules:**
```
- Extract only completed projects
- Focus on decisions from the last quarter
- Include confidence scores for each extraction
```

3. **Adjust output format:**
```json
{
  "entities": {
    "people": [...],
    "projects": [...]
  },
  "relationships": [
    {"from": "p001", "to": "pr001", "type": "WORKED_ON"}
  ]
}
```

## Handling Multiple Documents

The workflow processes documents **sequentially**. For each document:

1. Entities are extracted independently
2. IDs are generated per-document (p001, p002, etc.)
3. **Duplicate detection** happens in Neo4j via `MERGE`

To merge entities across documents, use consistent naming:
- "Sarah Chen" (good)
- "S. Chen" vs "Sarah Chen" (will create duplicates)

## Option 2: Hybrid Approach

Combine both workflows:

1. **Structured data** → Use `n8n_ingestion_fixed_v2.json`
2. **Unstructured text** → Use `n8n_ingestion_from_text.json`
3. Neo4j `MERGE` will deduplicate entities

## Option 3: Advanced NER (Named Entity Recognition)

For production use, consider:

**spaCy NER:**
- More accurate entity extraction
- Requires Python service
- Can extract: PERSON, ORG, DATE, GPE, PRODUCT

**LangChain Graph Transformers:**
- Built-in graph construction from text
- Supports multiple LLMs
- Can use n8n's LangChain nodes

**Neo4j NLP:**
- Native Neo4j NLP capabilities
- APOC NLP procedures
- Integrates with Google NLP, AWS Comprehend

## Troubleshooting

**LLM returns invalid JSON:**
- Check the "Parse Extracted Data" node output
- The code removes markdown fences automatically
- Adjust temperature to 0 for more consistent output

**Entities not linking correctly:**
- Ensure IDs are consistent (p001, pr001, etc.)
- Check that relationship fields match (approved_by, project_id)
- Verify Neo4j constraints are created

**Missing entities:**
- Make your text more explicit about relationships
- Add examples to the LLM prompt
- Increase LLM context window if text is long

**Duplicate entities:**
- Use consistent naming in source text
- Add deduplication logic in "Parse Extracted Data" node
- Consider adding entity resolution step

## Next Steps

1. **Test with sample data** - Use `sample_meeting_notes.txt`
2. **Refine the prompt** - Adjust extraction rules for your domain
3. **Add embeddings** - Copy embedding nodes from `n8n_ingestion_fixed_v2.json`
4. **Query the graph** - Use the agent workflow to ask questions

## Example Queries After Ingestion

```cypher
// Find all decisions from meeting notes
MATCH (d:Decision)
WHERE d.id STARTS WITH 'd'
RETURN d.title, d.date, d.approved_by

// Find projects mentioned in text
MATCH (p:Project)-[:USED_TOOL]->(t:Tool)
RETURN p.name, collect(t.name) AS tools

// Find who worked on what
MATCH (person:Person)-[:WORKED_ON]->(project:Project)
RETURN person.name, project.name
```

## Performance Considerations

- **LLM calls are slow** - Each document takes 5-30 seconds
- **Cost** - Groq is free (rate limited), OpenAI charges per token
- **Accuracy** - LLMs can hallucinate, always validate critical data
- **Batch processing** - Process multiple documents in parallel if needed

## Sample Files

- `sample_meeting_notes.txt` - Example meeting notes with entities
- `n8n_ingestion_from_text.json` - Complete workflow
- `README_TEXT_INGESTION.md` - This guide
