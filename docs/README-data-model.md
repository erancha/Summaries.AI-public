# Data Model

This document describes the database schema for Summaries.AI. The system uses a hybrid database approach with PostgreSQL for relational data and DynamoDB for document storage.

<!-- toc -->

- [PostgreSQL Tables](#postgresql-tables)
  * [1. LLM Profiles Table](#1-llm-profiles-table)
  * [2. LLM Prompts Table](#2-llm-prompts-table)
  * [3. SaaS Tenants Table](#3-saas-tenants-table)
  * [4. Records Table](#4-records-table)
  * [5. Subrecords Table](#5-subrecords-table)
  * [6. Records User Access Table](#6-records-user-access-table)
  * [7. Usage Metrics Table](#7-usage-metrics-table)
  * [8. LLM Pricing Table](#8-llm-pricing-table)
- [DynamoDB Tables](#dynamodb-tables)
  * [1. Summaries Table](#1-summaries-table)
- [Relationships](#relationships)

<!-- tocstop -->

## PostgreSQL Tables

PostgreSQL is chosen for:
- **Normalized Relations**: Junction tables like records_user_access manage many-to-many relationships between users and shared records
- **ACID Transactions**: Multi-step operations like creating records with user access use explicit BEGIN/COMMIT/ROLLBACK transactions
- **Foreign Key Constraints**: Cascading deletes (ON DELETE CASCADE) automatically clean up related data when users or records are deleted
- **Complex Queries**: Support for advanced SQL operations like dynamic WHERE clauses, aggregations, and time-based bucketing for analytics

### 1. LLM Profiles Table

Stores configuration for different Large Language Model profiles that users can select from.

- **profile_id** (PK) - Unique identifier for each profile (UUID)
- **model_id** - Model identifier (e.g., 'gpt-4.1', 'anthropic.claude-3-5-sonnet-20240620-v1:0')
- **description** - Human-readable description of the profile
- **level** - Profile complexity/capability level
- **disabled** - Whether the profile is disabled (boolean, default: false)
- **priority** - Display order priority (integer, default: 1)
- **notes** - Additional notes about the profile
- **created_at** - Timestamp when the profile was created
- **updated_at** - Timestamp when the profile was last modified

### 2. LLM Prompts Table

Normalized storage for system and user prompts, separated from profiles for reusability.

- **prompt_id** (PK) - Unique identifier for each prompt (UUID)
- **name** - Descriptive name for the prompt
- **user_prompt** - User-facing prompt template
- **system_prompt** - System-level instructions for the LLM
- **created_at** - Timestamp when the prompt was created
- **updated_at** - Timestamp when the prompt was last modified

### 3. SaaS Tenants Table

Multi-tenant user management with billing and contact information.

- **saas_tenant_id** (PK) - Unique identifier for each tenant/user (UUID)
- **disabled** - Whether the account is disabled (boolean, default: false)
- **credit** - Available credit balance in dollars (integer, default: 0)
- **email** - User's email address (varchar 100, optional)
- **name** - User's full name (varchar 255, optional)
- **phone** - Phone number (varchar 30, optional)
- **address** - Physical address (varchar 200, optional)
- **israeli_id** - Israeli ID number (varchar 9, optional, with regex validation for 8-9 digits)
- **created_at** - Account creation timestamp
- **updated_at** - Last modification timestamp

### 4. Records Table

Main conversation records between users and AI, representing complete chat sessions.

- **record_id** (PK) - Unique identifier for each record (UUID)
- **user_text** - User's input text
- **ai_text** - AI-generated response text
- **subject** - Extracted subject/title from the conversation
- **llm_profile_id** (FK) - References llm_profiles(profile_id), nullable
- **llm_prompt_id** (FK) - References llm_prompts(prompt_id), nullable
- **is_public** - Whether the record is publicly accessible (boolean, default: false)
- **is_archived** - Whether the record is archived (boolean, default: false)
- **saas_tenant_id** (FK) - References saas_tenants(saas_tenant_id), cascade delete
- **created_at** - Record creation timestamp
- **updated_at** - Last modification timestamp

### 5. Subrecords Table

Follow-up messages within a conversation record, enabling multi-turn dialogues.

- **subrecord_id** (PK) - Unique identifier for each subrecord (UUID)
- **record_id** (FK) - References records(record_id), cascade delete
- **user_text** - User's follow-up message
- **ai_text** - AI's response (nullable when non_ai is true)
- **non_ai** - Flag indicating no AI processing needed (boolean, default: false)
- **is_consolidated** - Whether this subrecord consolidates previous messages (boolean, default: false)
- **saas_tenant_id** (FK) - References saas_tenants(saas_tenant_id), cascade delete
- **created_at** - Subrecord creation timestamp

### 6. Records User Access Table

Junction table managing shared access to records between users.

- **id** (PK) - Auto-incrementing primary key
- **record_id** (FK) - References records(record_id), cascade delete
- **user_id** (FK) - References saas_tenants(saas_tenant_id), user with access
- **shared_by_user_id** (FK) - References saas_tenants(saas_tenant_id), user who granted access
- **created_at** - Access grant timestamp
- **Unique constraint** - Ensures one access record per (record_id, user_id) pair

### 7. Usage Metrics Table

Tracks LLM usage for billing and analytics purposes.

- **usage_id** (PK) - Unique identifier for each usage record (UUID)
- **saas_tenant_id** (FK) - References saas_tenants(saas_tenant_id), cascade delete
- **llm_profile_id** (FK) - References llm_profiles(profile_id), nullable
- **prompt_tokens** - Number of input tokens used (integer, default: 0)
- **completion_tokens** - Number of output tokens generated (integer, default: 0)
- **reasoning_tokens** - Number of reasoning tokens used (integer, default: 0)
- **cost** - Calculated cost in dollars (decimal 10,6, default: 0)
- **created_at** - Usage timestamp

### 8. LLM Pricing Table

Pricing configuration for different LLM models.

- **model_id** (PK) - Model identifier (varchar 255)
- **input_cost_per_million** - Cost per million input tokens (decimal 10,6)
- **output_cost_per_million** - Cost per million output tokens (decimal 10,6)
- **created_at** - Pricing entry creation timestamp
- **updated_at** - Last pricing update timestamp

## DynamoDB Tables

DynamoDB is chosen for:
- **Serverless AWS integration**: Native connectivity with chat-summarizer Lambda and S3 document processing with zero database maintenance
- **High-performance access**: Single-digit millisecond latency for summary retrieval that scales consistently
- **Flexible schema**: NoSQL structure allows for varying summary attributes without schema migrations
- **Global Secondary Indexes**: Efficient querying by record_id with automatic sorting by creation time

### 1. Summaries Table

Stores processed summaries of conversations for knowledge management and search.

- **id** (Partition Key) - Unique identifier for each summary (UUID)
- **record_id** - Associated record identifier (indexed via GSI)
- **subject** - Extracted subject/title from the content
- **content** - Human-readable summary content
- **summary** - LLM-optimized summary for processing
- **direction** - Text direction ('ltr' or 'rtl', default: 'ltr')
- **saas_tenant_id** - Tenant identifier for multi-tenancy 
- **created_at** - Summary creation timestamp

**Global Secondary Index**: RecordCreatedIndex
- Partition Key: record_id
- Sort Key: created_at
- Enables efficient querying of summaries by record with time-based sorting

**Note**: The `createSummary` function accepts a complete item object with all fields, allowing flexible summary creation with any combination of the above attributes.

## Relationships

- **Users (SaaS Tenants)** can create multiple **Records** (one-to-many)
- **Records** can have multiple **Subrecords** for follow-up conversations (one-to-many)
- **Records** can have multiple **Summaries** for knowledge management (one-to-many, stored in DynamoDB)
- **Records** can be shared with multiple **Users** via Records User Access junction table (many-to-many)
- **LLM Profiles** can be used by multiple **Records** (one-to-many)
- **LLM Prompts** can be used by multiple **Records** (one-to-many)
- **Usage Metrics** track consumption per **User** and **LLM Profile** (many-to-one relationships)
- **LLM Pricing** provides cost calculation data for **Usage Metrics** (one-to-many via model_id)

**Key Design Patterns:**
- **Soft Deletes**: Records use `is_archived` flag instead of hard deletion to preserve data integrity
- **Tenant Isolation**: All major entities include `saas_tenant_id` for multi-tenant data separation
- **Audit Trail**: Comprehensive `created_at` and `updated_at` timestamps across all entities
- **Flexible Access Control**: Records can be private (owner only), shared (via access table), or public (globally accessible)

## Key Database Operations

### PostgreSQL Operations
- **Transaction Management**: Record creation automatically includes user access entry in a single transaction
- **Dynamic Queries**: Usage metrics support flexible filtering by tenant, profile, and date ranges
- **Aggregation Functions**: Built-in analytics for token usage, costs, and time-based bucketing
- **Search Optimization**: Partial indexes on `is_public` and `is_archived` for efficient filtering
- **Relationship Management**: Automatic cleanup via foreign key constraints with CASCADE deletes

### DynamoDB Operations
- **Batch Operations**: Efficient summary retrieval with optional limits and pagination
- **Conditional Writes**: Summary deletion includes tenant validation for security
- **Index Queries**: RecordCreatedIndex enables fast lookup of summaries by record with time sorting
- **Flexible Schema**: Summary creation accepts any valid item structure for extensibility
