# Database Entities (MVP)

## Overview

This document outlines the database structure and relationships for the labXpert.ai conversational AI system's first version (MVP). Each entity is designed to support multi-level organization, workspace isolation, conversational recipe management, proactive safety, standard enforcement, education, and organizational memory access. In this MVP, organizations and workspaces are pre-created by developers.

## Entities

### 1. Organizations

- **Purpose**: Top-level entity representing a company, institution, or group that contains multiple workspaces and folders.
- **Attributes**:
  - `id`: Unique identifier.
  - `name`: Organization name.
  - `description`: Brief overview of the organization.
  - `logo_url`: URL to the organization's logo.
  - `created_at`: Creation timestamp.
  - `configuration`: JSONB configuration for organization-level settings.
- **DDL**:

```sql
CREATE TABLE organizations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name TEXT NOT NULL,
    description TEXT,
    logo_url TEXT,
    created_at TIMESTAMP DEFAULT now(),
    configuration JSONB NOT NULL
);
```

- **RLS Policy Example (Membership-Based)**:

```sql
-- Enable RLS on organizations
ALTER TABLE organizations ENABLE ROW LEVEL SECURITY;

-- Policy: Allow only users who are active members to read organizations
CREATE POLICY "Members can read their organizations" ON organizations
  FOR SELECT
  USING (
    EXISTS (
      SELECT 1 FROM profile_organizations
      WHERE profile_organizations.organization_id = organizations.id
        AND profile_organizations.profile_id = auth.uid()
        AND profile_organizations.is_active = true
    )
  );
```

### 2. Workspaces

- **Purpose**: Define isolated environments for teams/departments within an organization, pre-created by developers.
- **Attributes**:
  - `id`: Unique identifier.
  - `organization_id`: Foreign key to the parent organization.
  - `name`: Workspace name.
  - `description`: Brief overview of the workspace purpose.
  - `custom_instructions`: Tailored instructions for the AI agent's behavior within this workspace.
  - `created_at`: Creation timestamp.
- **DDL**:

```sql
CREATE TABLE workspaces (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID REFERENCES organizations(id) NOT NULL,
    name TEXT NOT NULL,
    description TEXT,
    custom_instructions TEXT,
    created_at TIMESTAMP DEFAULT now()
);
```

- **RLS Policy Example (Membership-Based)**:

```sql
-- Enable RLS on workspaces
ALTER TABLE workspaces ENABLE ROW LEVEL SECURITY;

-- Policy: Allow only users who are active members to read workspaces
CREATE POLICY "Members can read their workspaces" ON workspaces
  FOR SELECT
  USING (
    EXISTS (
      SELECT 1 FROM profile_workspaces
      WHERE profile_workspaces.workspace_id = workspaces.id
        AND profile_workspaces.profile_id = auth.uid()
        AND profile_workspaces.is_active = true
    )
  );
```

### 3. Folders

- **Purpose**: Organize resources within an organization (documents, templates, etc.).
- **Attributes**:
  - `id`: Unique identifier.
  - `organization_id`: Foreign key to the parent organization.
  - `parent_folder_id`: Self-reference for nested folders (NULL for top-level folders).
  - `name`: Folder name.
  - `description`: Optional description of the folder's contents.
  - `created_at`: Creation timestamp.
- **DDL**:

```sql
CREATE TABLE folders (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID REFERENCES organizations(id) NOT NULL,
    parent_folder_id UUID REFERENCES folders(id),
    name TEXT NOT NULL,
    description TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

- **RLS Policy Example (Membership-Based)**:

```sql
-- Enable RLS on folders
ALTER TABLE folders ENABLE ROW LEVEL SECURITY;

-- Policy: Allow only users who are active members of the organization to read folders
CREATE POLICY "Members can read their organization's folders" ON folders
  FOR SELECT
  USING (
    EXISTS (
      SELECT 1 FROM profile_organizations
      WHERE profile_organizations.organization_id = folders.organization_id
        AND profile_organizations.profile_id = auth.uid()
        AND profile_organizations.is_active = true
    )
  );
```

### 4. Profiles

- **Purpose**: Store application-specific user data and track creators/editors within the conversational agent.
- **Attributes**:
  - `id`: Primary key that matches the auth.users id (direct 1:1 relationship).
  - `first_name`: User's first name.
  - `last_name`: User's last name.
  - `email`: User's email (synchronized with auth.users).
  - `created_at`: Timestamp with time zone.
  - `updated_at`: Last update timestamp with time zone.
- **DDL**:

```sql
CREATE TABLE profiles (
    id UUID PRIMARY KEY,  -- Same as auth.users.id
    first_name TEXT NOT NULL,
    last_name TEXT,
    email TEXT NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT now(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT now()
);
```

- **RLS Policy Example (Self-Access)**:

```sql
-- Enable RLS on profiles
ALTER TABLE profiles ENABLE ROW LEVEL SECURITY;

-- Policy: Users can read their own profile
CREATE POLICY "Users can read their own profile" ON profiles
  FOR SELECT
  USING (
    id = auth.uid()
  );
```

### 5. Profile Organizations

- **Purpose**: Enable many-to-many relationship between profiles and organizations, allowing users to belong to multiple organizations.
- **Attributes**:
  - `id`: Unique identifier.
  - `profile_id`: Foreign key to the profile.
  - `organization_id`: Foreign key to the organization.
  - `role`: User's role within the organization (e.g., "admin", "member").
  - `is_active`: Flag indicating if the membership is active.
  - `created_at`: Timestamp when the relationship was created.
- **DDL**:

```sql
CREATE TABLE profile_organizations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    profile_id UUID REFERENCES profiles(id) NOT NULL,
    organization_id UUID REFERENCES organizations(id) NOT NULL,
    role TEXT NOT NULL DEFAULT 'member',
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT now(),
    UNIQUE(profile_id, organization_id)
);

CREATE INDEX idx_profile_organizations_profile_id ON profile_organizations(profile_id);
CREATE INDEX idx_profile_organizations_organization_id ON profile_organizations(organization_id);
```

- **RLS Policy Example (Self-Access)**:

```sql
-- Enable RLS on profile_organizations
ALTER TABLE profile_organizations ENABLE ROW LEVEL SECURITY;

-- Policy: Users can read their own organization memberships
CREATE POLICY "Users can read their own organization memberships" ON profile_organizations
  FOR SELECT
  USING (
    profile_id = auth.uid()
  );
```

### 6. Profile Workspaces

- **Purpose**: Enable many-to-many relationship between profiles and workspaces, allowing users to belong to multiple workspaces.
- **Attributes**:
  - `id`: Unique identifier.
  - `profile_id`: Foreign key to the profile.
  - `workspace_id`: Foreign key to the workspace.
  - `role`: User's role within the workspace (e.g., "admin", "member").
  - `is_active`: Flag indicating if the membership is active.
  - `created_at`: Timestamp when the relationship was created.
- **DDL**:

```sql
CREATE TABLE profile_workspaces (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    profile_id UUID REFERENCES profiles(id) NOT NULL,
    workspace_id UUID REFERENCES workspaces(id) NOT NULL,
    role TEXT NOT NULL DEFAULT 'member',
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT now(),
    UNIQUE(profile_id, workspace_id)
);

CREATE INDEX idx_profile_workspaces_workspace_id ON profile_workspaces(workspace_id);
CREATE INDEX idx_profile_workspaces_profile_id ON profile_workspaces(profile_id);
```

- **RLS Policy Example (Self-Access)**:

```sql
-- Enable RLS on profile_workspaces
ALTER TABLE profile_workspaces ENABLE ROW LEVEL SECURITY;

-- Policy: Users can read their own workspace memberships
CREATE POLICY "Users can read their own workspace memberships" ON profile_workspaces
  FOR SELECT
  USING (
    profile_id = auth.uid()
  );
```

### 7. Conversations

- **Purpose**: Log conversations for auditing, debugging, agent improvement, and as a source for advanced knowledge retrieval about past discussions and decisions.
- **Attributes**:
  - `id`: Unique identifier for the conversation.
  - `workspace_id`: Link to Workspace (data isolation).
  - `messages`: JSONB array containing the entire conversation history.
  - `created_at`: Creation timestamp.
  - `profile_id`: Link to the user's profile.
  - `is_canvas_created`: Flag indicating if a canvas visualization was created.
  - `agent`: Type of AI agent used (e.g., 'RECIPE_GENERATOR').
  - `title`: Optional conversation title for easy reference.
  - `canvas_content`: Optional storage for chemical structure visualizations.
  - `organization_id`: Foreign key to the parent organization.
- **DDL**:

```sql
CREATE TABLE conversations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id UUID REFERENCES workspaces(id) NOT NULL,
    messages JSONB NOT NULL DEFAULT '[]'::jsonb,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT now(),
    profile_id UUID NOT NULL REFERENCES profiles(id),
    is_canvas_created BOOLEAN NOT NULL DEFAULT false,
    agent TEXT NOT NULL DEFAULT 'RECIPE_GENERATOR'::text,
    title TEXT NULL,
    canvas_content TEXT NULL,
    organization_id UUID REFERENCES organizations(id) NOT NULL
);
```

- **Message Structure Example**:

```json
[
  {
    "role": "user",
    "content": "Create a recipe for aspirin synthesis",
    "timestamp": "2023-07-15T14:30:22Z"
  },
  {
    "role": "assistant",
    "content": "I'll help you create an aspirin synthesis recipe. What scale would you like to produce?",
    "timestamp": "2023-07-15T14:30:25Z"
  }
]
```

- **RLS Policy Example (Membership-Based)**:

```sql
-- Enable RLS on conversations
ALTER TABLE conversations ENABLE ROW LEVEL SECURITY;

-- Policy: Allow only users who are active members of the workspace or organization to read, update, insert, or delete conversations
CREATE POLICY "Members can read their workspace's conversations" ON conversations
  FOR SELECT
  TO authenticated
  USING (
    ((workspace_id IS NOT NULL AND EXISTS (
      SELECT 1 FROM profile_workspaces pw
      WHERE pw.workspace_id = conversations.workspace_id
        AND pw.profile_id = auth.uid()
        AND pw.is_active
    )) OR (workspace_id IS NULL AND EXISTS (
      SELECT 1 FROM profile_organizations po
      WHERE po.organization_id = conversations.organization_id
        AND po.profile_id = auth.uid()
        AND po.is_active
    )))
  );
CREATE POLICY "Members can update their workspace's conversations" ON conversations
  FOR UPDATE
  TO authenticated
  USING (
    ((workspace_id IS NOT NULL AND EXISTS (
      SELECT 1 FROM profile_workspaces pw
      WHERE pw.workspace_id = conversations.workspace_id
        AND pw.profile_id = auth.uid()
        AND pw.is_active
    )) OR (workspace_id IS NULL AND EXISTS (
      SELECT 1 FROM profile_organizations po
      WHERE po.organization_id = conversations.organization_id
        AND po.profile_id = auth.uid()
        AND po.is_active
    )))
  )
  WITH CHECK (
    ((workspace_id IS NOT NULL AND EXISTS (
      SELECT 1 FROM profile_workspaces pw
      WHERE pw.workspace_id = conversations.workspace_id
        AND pw.profile_id = auth.uid()
        AND pw.is_active
    )) OR (workspace_id IS NULL AND EXISTS (
      SELECT 1 FROM profile_organizations po
      WHERE po.organization_id = conversations.organization_id
        AND po.profile_id = auth.uid()
        AND po.is_active
    )))
  );
CREATE POLICY "Members can insert to their workspace's conversations" ON conversations
  FOR INSERT
  TO authenticated
  WITH CHECK (
    ((workspace_id IS NOT NULL AND EXISTS (
      SELECT 1 FROM profile_workspaces pw
      WHERE pw.workspace_id = conversations.workspace_id
        AND pw.profile_id = auth.uid()
        AND pw.is_active
    )) OR (workspace_id IS NULL AND EXISTS (
      SELECT 1 FROM profile_organizations po
      WHERE po.organization_id = conversations.organization_id
        AND po.profile_id = auth.uid()
        AND po.is_active
    )))
  );
CREATE POLICY "Members can delete their workspace's conversations" ON conversations
  FOR DELETE
  TO authenticated
  USING (
    ((workspace_id IS NOT NULL AND EXISTS (
      SELECT 1 FROM profile_workspaces pw
      WHERE pw.workspace_id = conversations.workspace_id
        AND pw.profile_id = auth.uid()
        AND pw.is_active
    )) OR (workspace_id IS NULL AND EXISTS (
      SELECT 1 FROM profile_organizations po
      WHERE po.organization_id = conversations.organization_id
        AND po.profile_id = auth.uid()
        AND po.is_active
    )))
  );
```

### 8. Files

- **Purpose**: Store files related to conversations, recipes, and other resources.
- **Attributes**:
  - `id`: Unique identifier.
  - `created_at`: Creation timestamp.
  - `organization_id`: Foreign key to the parent organization.
  - `workspace_id`: Link to Workspace (data isolation).
  - `profile_id`: Reference to the profile that owns this file.
  - `folder_id`: Reference to folder containing this file.
  - `name`: File name.
  - `description`: Optional description of the file.
  - `file_link`: URL to the file.
  - `signed_url`: URL for accessing the file.
  - `bucket_path`: Path to file in storage bucket.
  - `document_type`: Type of document (e.g., PDF, spreadsheet).
  - `status`: Processing status of the file.
  - `openai_file_id`: ID of file in OpenAI system if uploaded.
  - `openai_vector_store_id`: Reference to vector store containing embeddings.
  - `metadata`: Additional file metadata in JSON format.
- **DDL**:

```sql
CREATE TABLE files (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT now(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    workspace_id UUID REFERENCES workspaces(id) NOT NULL,
    profile_id UUID NOT NULL REFERENCES profiles(id),
    folder_id UUID REFERENCES folders(id),
    name TEXT NOT NULL,
    description TEXT,
    file_link TEXT NOT NULL,
    signed_url TEXT NOT NULL,
    bucket_path TEXT,
    document_type TEXT,
    status TEXT,
    openai_file_id TEXT,
    openai_vector_store_id TEXT REFERENCES vector_stores(openai_vector_store_id),
    metadata JSONB
);

CREATE INDEX idx_files_profile_id ON files(profile_id);
CREATE INDEX idx_files_folder_id ON files(folder_id);
```

- **RLS Policy Example (Membership-Based)**:

```sql
-- Enable RLS on files
ALTER TABLE files ENABLE ROW LEVEL SECURITY;

-- Policy: Allow only users who are active members of the organization to read files
CREATE POLICY "Members can read their organization's files" ON files
  FOR SELECT
  USING (
    EXISTS (
      SELECT 1 FROM profile_organizations
      WHERE profile_organizations.organization_id = files.organization_id
        AND profile_organizations.profile_id = auth.uid()
        AND profile_organizations.is_active = true
    )
  );
```

### 9. Vector Stores

- **Purpose**: Store information about vector embeddings collections used for semantic search and retrieval.
- **Attributes**:
  - `id`: Unique identifier.
  - `openai_vector_store_id`: External ID in OpenAI service.
  - `created_at`: Creation timestamp.
  - `name`: Name of the vector store.
  - `description`: Optional description.
  - `metadata`: Additional metadata in JSON format.
  - `organization_id`: Foreign key to the parent organization.
  - `workspace_id`: Foreign key to the workspace.
- **DDL**:

```sql
CREATE TABLE vector_stores (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    openai_vector_store_id TEXT UNIQUE NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT now(),
    name TEXT NOT NULL,
    description TEXT,
    metadata JSONB,
    organization_id UUID REFERENCES organizations(id),
    workspace_id UUID REFERENCES workspaces(id)
);
```

- **RLS Policy Example (Membership-Based)**:

```sql
-- Enable RLS on vector_stores
ALTER TABLE vector_stores ENABLE ROW LEVEL SECURITY;

-- Policy: Allow only users who are active members of the organization to read vector stores
CREATE POLICY "Members can read their organization's vector stores" ON vector_stores
  FOR SELECT
  USING (
    EXISTS (
      SELECT 1 FROM profile_organizations
      WHERE profile_organizations.organization_id = vector_stores.organization_id
        AND profile_organizations.profile_id = auth.uid()
        AND profile_organizations.is_active = true
    )
  );
```

### 10. Tool Conversation

- **Purpose**: Log tool calls associated with conversations for audit, traceability, or agent actions.
- **Attributes**:
  - `tool_call_id`: Unique identifier for the tool call (text, PK).
  - `conversation_id`: Foreign key to the conversation (uuid).
  - `created_at`: Timestamp when the tool call was logged.
  - `tool_name`: Name of the tool invoked.
  - `data`: JSONB payload with tool call details.
- **DDL**:

```sql
CREATE TABLE tool_conversation (
    tool_call_id TEXT PRIMARY KEY,
    conversation_id UUID NOT NULL REFERENCES conversations(id),
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT now(),
    tool_name TEXT NOT NULL,
    data JSONB
);
```

- **RLS Policy Example (Membership-Based)**:

```sql
-- Enable RLS on tool_conversation
ALTER TABLE tool_conversation ENABLE ROW LEVEL SECURITY;

-- Policy: Allow only users who are active members of the workspace to read tool calls
CREATE POLICY "Members can read their workspace's tool calls" ON tool_conversation
  FOR SELECT
  USING (
    EXISTS (
      SELECT 1 FROM conversations
      JOIN profile_workspaces ON conversations.workspace_id = profile_workspaces.workspace_id
      WHERE conversations.id = tool_conversation.conversation_id
        AND profile_workspaces.profile_id = auth.uid()
        AND profile_workspaces.is_active = true
    )
  );
```

## Relationships

- **Organizations ↔ Workspaces**: One-to-many (one organization contains many workspaces).
- **Organizations ↔ Folders**: One-to-many (one organization contains many folders).
- **Folders ↔ Folders**: Self-referential for nested folders (parent-child relationship).
- **Folders ↔ Files**: One-to-many (folders can contain multiple files).
- **Profiles ↔ Auth Users**: One-to-one (profiles.id = auth.users.id for a direct relationship).
- **Profiles ↔ Organizations**: Many-to-many via `profile_organizations` table (profiles can belong to multiple organizations with different roles).
- **Profiles ↔ Primary Workspace**: Many-to-one (legacy: each profile was assigned to exactly one workspace through workspace_id).
- **Profiles ↔ Workspaces**: Many-to-many via `profile_workspaces` table (profiles can belong to multiple workspaces with different roles).
- **Profiles ↔ Files**: One-to-many (users can own multiple files).
- **Workspaces ↔ All Data Entities**: One-to-many (each workspace contains its own isolated set of conversations, files, etc.).
- **Vector Stores ↔ Files**: One-to-many (embeddings collection can be associated with multiple files).
- **Conversations ↔ Tool Conversation**: One-to-many (each conversation can have multiple tool call logs).
