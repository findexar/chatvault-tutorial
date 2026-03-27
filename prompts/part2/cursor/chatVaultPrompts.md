Title: ChatVault Backend – Apps SDK / MCP Vibe Engineering PROMPTS

project name - chatvault-part2

This project uses the **generic backend MCP server prompts** defined in:

- `prompts/part2/cursor/common.md`

Use that file for:

- **Prompt1**: Setup Neon PostgreSQL Database
- **Prompt2**: Initialize Node.js Project with Drizzle + Apps SDK, deploy
- **Prompt3**: Install Dependencies + Initialize Drizzle
- **Prompt4**: Create Basic MCP HTTP Streaming Server
- **Prompt5**: Setup Generic Test Framework

This file defines the **ChatVault-specific backend behavior** starting from Prompt6.

## Engineering Principles (ChatVault-specific)

- **Align with the generic prompts**: All work here inherits the engineering principles from `common.md` (verify, test with real databases, graceful degradation, separate concerns). Do not introduce project-specific shortcuts that violate those principles.
- **Maintain Part 1 compatibility**: The Part 1 widget calls the MCP tool **`loadMyChats`** (not `loadChats`). Its `structuredContent` must match Part 1: **`{ chats: [...], nextCursor: string | null }`**, with **cursor-based** pagination (pass `cursor` from the previous response to fetch the next page). Part 1’s reference server does **not** attach `_meta` to `loadMyChats` results; **`_meta` with `chatVault` / UI hints** applies to **`browseMySavedChats`**, not to listing chats. Part 2 may include extra fields on each chat (e.g. `id`) as a superset of the Part 1 shape.
- **Design for observability**: All database operations should be logged (queries, results, errors). Use structured logging where possible to make debugging easier.
- **Vector search quality**: When implementing vector search, test with various query types (short, long, technical terms, natural language) to ensure embeddings capture semantic meaning correctly.

---

Prompt6: Implement `saveChat` Tool

Define the Chat schema in Drizzle with fields for id, userId, title, timestamp, turns (as JSONB), and embedding (vector type). Create an embeddings utility that can generate vector embeddings for text—use OpenAI's Embeddings API. Implement the `saveChat` MCP tool that takes userId, title, and turns as parameters, generates an embedding for the entire chat (combining all prompts and responses), and saves it to the database. Register the tool in the MCP server and add comprehensive logging.

**Non-negotiables:**

- `userId` must be a required parameter (not optional)
- Embedding must be generated for the entire chat (all prompts + responses combined)
- All errors must be caught and returned as JSON-RPC errors
- Tool must return chat ID in response for reference

---

Prompt7: Implement `loadMyChats` Tool

Implement the **`loadMyChats`** MCP tool (same name as Part 1) that retrieves paginated chat data from PostgreSQL. Parameters: **`userId`** (required), **`cursor`** (optional, opaque string from the previous page’s `nextCursor`), **`limit`** (optional, default 10, cap as appropriate). Query chats for that `userId`, ordered by **timestamp descending** (then stable tie-break, e.g. `id` descending). Use **keyset (cursor) pagination**: return up to `limit` rows and, if more exist, a non-null **`nextCursor`** for the next request; otherwise `nextCursor: null`. Return **`structuredContent: { chats, nextCursor }`** plus human-readable `content`, matching Part 1’s reference server. Handle **empty results** (`chats: []`, `nextCursor: null`) and **invalid cursor** errors without crashing the server.

**Non-negotiables:**

- Tool name and `structuredContent` shape must match Part 1: **`loadMyChats`** and **`{ chats, nextCursor }`** (same field names Part 1 uses)
- Do **not** require `_meta` on the load tool result for Part 1 parity (browse/bootstrap is separate)

---

Prompt8: Implement `searchMyChats` Tool (Vector Search)

Implement the **`searchMyChats`** MCP tool (Part 2 name; aligns with `loadMyChats`) that performs vector similarity search on chat embeddings. Create a vector search query function that uses pgvector’s cosine similarity operator to find chats matching a query embedding. Parameters: **`userId`** (required), **`query`** (required), **`limit`** (optional, default 10). Generate an embedding for the search query, run similarity search scoped to that user, return results **ordered by similarity (most similar first)**. Format the response similarly to **`loadMyChats`** (e.g. a `chats` array) but include **search-specific metadata** (e.g. similarity scores). Handle chats without embeddings by **excluding** them from results (not an error).

**Non-negotiables:**

- `userId` and `query` must be required parameters
- Search must only return chats that belong to the specified `userId`
- Search must only return chats with non-null embeddings
- Results must be ordered by similarity (most similar first)
- Default `limit` must be 10
- Must handle empty results gracefully (not an error)

---

Prompt9: Update Tests for ChatVault Actions

Add comprehensive end-to-end tests for **`saveChat`**, **`loadMyChats`**, and **`searchMyChats`** using a real test database. Cover success paths, missing/invalid arguments, and edge cases (empty lists, **cursor pagination**). Include integration flows (**save → load → search**). Helpers for DB setup/teardown and **`tools/list`** assertions should include these three tool names.

**Non-negotiables:**

- All tests must use real database (local PostgreSQL)
- All tests must clean up after themselves (no leftover data)
- Tests must verify database state, not just API responses
- Tests must cover error cases (missing params, invalid data)
- Tests must verify response formats match Part 1 exactly
- All tests must be independent (can run in any order)

---

Prompt10: ChatGPT Integration and Testing

Start the backend server, set up an ngrok tunnel, and configure ChatGPT to connect to the MCP server. Ensure production DB is up-to-date with migrations. Test all three tools (`saveChat`, `loadMyChats`, `searchMyChats`) from ChatGPT with various prompts. Verify error handling works correctly and that responses are clear and actionable. Optionally test integration with the Part 1 widget if available. Document the integration steps for future reference.

**Non-negotiables:**

- All three tools must work from ChatGPT
- Error messages must be clear and actionable
- Database operations must be visible in server logs
- Integration must be documented for future reference

---

**Next Steps:**

After completing Prompt10, the ChatVault backend is complete and ready for:

- Integration with Part 1 widget (if desired)
- Production deployment (Part 4)
- SaaS layer integration (Part 3)
