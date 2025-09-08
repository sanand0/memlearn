# Initial Prompt, 2025-09-9 1:30 pm SGT

Goal: A **static** single-page web app (no backend, **localStorage only**) that (1) ingests documents, (2) extracts **atomic memory items** (type=`code|llm`), (3) consolidates (delete/merge/edit), and (4) **validates** new documents using those memories.

- Ingests user-uploaded documents (**.txt, .md, .pdf**) in user-configurable batches (**default: 5**) to **extract** memory items (in code or llm)

  - Pass only 1 document to LLM per request. Run batches in parallel.
  - Pass PDF files base64-encoded; pass text files as text.
  - LLM generates a memory object like this using a JSON schema response_format:
    ```js
    {
      "type": "code" | "llm",
      "rule": "code: JS function source; llm: instruction for how to check",
      "priority": "low" | "medium" | "high",
      "rationale": "why this memory/rule matters",
      "sources": [
        {"quote": "..."}
      ],
    }
    ```
  - The JS code generated should be a `function check(doc) { ... return { pass: true/false, reason: "..." } }`n
  - App adds metadata:
    - id: very short, unique string
    - updates each item in sources to add `filename`:

- consolidates memory (delete/merge/edit) if memory count > user-configurable threshold (default: 30) or on user clicking on "Consolidate" button. Consolidation pauses ingestion.
  - OpenAI model generates consolidation suggestions passing the LLM a JSON schema:
    ```js
    {
      "deletes": [{ "id": "...", "reason": "duplicate of mem_y" }],
      "edits": [{ "id": "...", "update": { "rule": "…", "priority": 2, "rationale": "…" }, "reason": "clarify scope" }],
      "merges": [
        {
          "from": ["...", "..."],  // 2+ IDs to merge
          "update": { // merged
            "type": "code|llm",
            "rule": ...
            "priority": ...,
            "rationale": ...
          },
          "reason": "generalized rules",
          // app adds new unique ID for merged rule.
          // app merges sources: by concatenating sources
        }
      ]
    }
    ```
  - app applies suggestions and consolidates memory list

This app is a demo of this workflow. Show progress continuously. Specifically:

- As soon as there is an LLM fetch triggered, show a progress indicator
- Show the status of each document as an elegant list, e.g. "queued", "processing", "done", "error: ..."
- Stream the memory items and show them as they get extracted using lit-html.
- Stream **per-document** results (batch-level streaming). Parse via partial-json and render in real-time.
- Alongside the document status, show the memory list as it gets populated and updated.
- At the bottom, show an elegant collapsible history of actions that shows every step, e.g.
  - Ingestion: For each document, show a summary of the memory items extracted. The collapsed view will show the documents, expanded view will show the memory items under each.
  - Consolidation: Show each consolidation step (deletes, edits, merges) at the top level; expanded view will show details, including reason

Follow the visual and coding style of /home/sanand0/code/datagen/ - same LICENSE, README structure, `index.html` scaffold, config shape, and code style as `script.js` from that project. This repo will be at https://github.com/sanand0/memlearn and deployed at https://sanand0.github.com/memlearn/

Use the OpenAI responses API. Here's an example:

```bash
curl $OPENAI_BASE_URL/v1/responses \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -d '{
    "model": "...",
    "instructions": "..."
    "input": [
      {
        "role": "user",
        "content": [
          {"type": "input_text", "text": "# filename.ext\n\n..."},
          // For PDFs:
          {"type": "input_file", "filename": "...", "file_data": "... base64 ..."}
        ]
      }
    ],
    "response_format": {
      "type": "json_schema",
      "strict": true,
      "json_schema": {
        "name": "person",
        "strict": true,
        "schema": { ... /* with required: [...], and additionalProperties: false */}
      }
    }
  }'
```
