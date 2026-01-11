# Data Model

**1. LLM profiles Table**:

- **profile_id** (PK) - Unique identifier for each profile (UUID).
- **description** - String. Profile's description.
- **modelId** - String. Model id (e.g. 'gpt-4.1-nano').
- **userPrompt** - String. User prompt.
- **systemPrompt** - String. System prompt.
- **created_at** - Timestamp when the profile was created.
- **updated_at** - Timestamp when the profile was last modified.

**2. SaaS Tenants Table**:

- **saas_tenant_id** (PK) - Identifier for SaaS multi-tenancy.
- **name**
- **email**
- **phone**
- **address**
- **disabled** - If disabled, the role1 account is no longer active.

**3. Chats Table**:

- **chat_id** (PK) - Unique identifier for each chat (UUID).
- **llm_profile_id** (FK) - Unique identifier of the LLM profile used by ths chat (UUID).
- **user_text** - String. User entered text.
- **ai_text** - String. AI generated text.
- **created_at** - Timestamp when the chat was created.
- **updated_at** - Timestamp when the chat was last modified.
- **saas_tenant_id** (Partition Key) - Identifier for SaaS multi-tenancy.

**4. ChatMessagess Table**:

- **messages_id** (PK) - Unique identifier for each chat message (UUID).
- **chat_id** - Identifier for the chat associated with the chat messages.
- **created_at** - Date when the chat message was made.
- **user_text** - String. Chat message's user entered text.
- **ai_text** - String. Chat message's AI generated text.
- **confirmed_at**: - Date of confirmation.
- **saas_tenant_id** (Partition Key) - Identifier for SaaS multi-tenancy.

**5. Chats_user_access Table**:

- **id** (PK) - Auto-incrementing primary key.
- **chat_id** (FK) - References Chats(chat_id), cascades on delete.
- **user_id** (FK) - References saas_tenants(saas_tenant_id), user who has access to the chat.
- **shared_by_user_id** (FK) - References saas_tenants(saas_tenant_id), user who shared the chat.
- **created_at** - Timestamp when access was granted (defaults to now()).
- **Unique constraint** - Ensures one access chat per (chat_id, user_id) pair.

**6. Summaries Table**:

- **id** (Partition Key) - Unique identifier for each summary (UUID).
- **chat_id** - Identifier for the chat associated with the summary.
- **pdf_url** - URL of the pdf in S3.
- **created_at** - Timestamp when the summary was created.
- **saas_tenant_id** (GSI Partition Key) - Identifier for SaaS multi-tenancy.

**Example of Relationships**:

- A user can have multiple chats.
- A chat can have multiple messages.
