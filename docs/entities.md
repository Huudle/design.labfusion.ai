# Database Entities (MVP)

## Overview

This document outlines the database structure and relationships for the labfusion.ai conversational AI system's first version (MVP). Each entity is designed to support workspace isolation, conversational recipe management, proactive safety, standard enforcement, education, and organizational memory access. In this MVP, workspaces are pre-created by developers.

## Entities

### 1. Workspaces

- **Purpose**: Define isolated environments for teams/organizations, pre-created by developers.
- **Attributes**:
  - `id`: Unique identifier.
  - `name`: Workspace name.
  - `description`: Brief overview of the workspace purpose.
  - `custom_instructions`: Tailored instructions for the AI agent's behavior within this workspace.
  - `agent_configurations`: JSONB field storing configurations for different agent types within the workspace.
  - `created_at`: Creation timestamp.
- **DDL**:

```sql
CREATE TABLE workspaces (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    description TEXT,
    custom_instructions TEXT,
    agent_configurations JSONB DEFAULT '{}'::jsonb,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

- **Agent Configurations Structure**:
  - JSONB object containing configuration for each agent type
  - Example:
  ```json
  {
    "workspace_agent": {
      "model": "gpt-4",
      "temperature": 0.2,
      "enabled_tools": ["web_search", "file_search", "recipe_tools"]
    },
    "recipe_agent": {
      "model": "gpt-4",
      "temperature": 0.1,
      "specialized_instructions": "Focus on pharmaceutical synthesis techniques...",
      "enabled_tools": ["web_search", "recipe_tools", "inventory_tools"]
    },
    "safety_agent": {
      "model": "gpt-4",
      "temperature": 0.0,
      "specialized_instructions": "Enforce FDA regulations and...",
      "enabled_tools": ["web_search", "sds_tools"]
    }
  }
  ```

### 2. Profiles

- **Purpose**: Store application-specific user data and track creators/editors within the conversational agent.
- **Attributes**:
  - `id`: Unique identifier.
  - `user_id`: Foreign key to Supabase auth.users table.
  - `name`: User's full name.
  - `email`: User's email (synchronized with auth.users).
  - `created_at`: Timestamp.
  - `assigned_workspace_id`: Assigned workspace (pre-configured by developers).
- **DDL**:

```sql
CREATE TABLE profiles (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES auth.users(id) NOT NULL,
    name VARCHAR(255) NOT NULL,
    email VARCHAR(255) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    assigned_workspace_id UUID REFERENCES workspaces(id) NOT NULL,
    UNIQUE(user_id)
);
```

### 3. Materials (Inventory)

- **Purpose**: Track materials for recipes, pricing, and safety data.
- **Attributes**:
  - `id`: Unique identifier.
  - `workspace_id`: Link to Workspace (data isolation).
  - `name`: Material name (e.g., "Acetone").
  - `category`: Type (e.g., "Reagent").
  - `quantity`: Amount in stock.
  - `unit`: Measurement unit (e.g., "mL").
  - `location`: Storage location.
  - `price`: Cost per unit.
  - `supplier`: Source of material.
  - `stock_date`: When added to inventory.
  - `expiration_date`: Expiry date (if applicable).
  - `sds_id`: Link to Material SDS.
- **DDL**:

```sql
CREATE TABLE materials (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id UUID REFERENCES workspaces(id) NOT NULL,
    name VARCHAR(255) NOT NULL,
    category VARCHAR(50),
    quantity DECIMAL NOT NULL,
    unit VARCHAR(20) NOT NULL,
    location VARCHAR(100),
    price DECIMAL NOT NULL,
    supplier VARCHAR(255),
    stock_date DATE,
    expiration_date DATE,
    sds_id UUID REFERENCES material_sds(id)
);
```

### 4. Material SDS

- **Purpose**: Store pre-existing safety data for materials, used by the agent for proactive checks, education, and AI-generated SDS.
- **Attributes**:
  - `id`: Unique identifier.
  - `workspace_id`: Link to Workspace (data isolation).
  - `material_id`: Link to Material.
  - `chemical_name`: Official name (e.g., "Acetone").
  - `hazards`: Safety risks (JSON for flexibility).
  - `handling`: Precautions.
  - `storage`: Storage instructions.
  - `first_aid`: Emergency measures.
  - `disposal`: Disposal guidelines.
  - `file_link`: URL to full SDS document.
  - `last_updated`: For tracking updates.
- **DDL**:

```sql
CREATE TABLE material_sds (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id UUID REFERENCES workspaces(id) NOT NULL,
    material_id UUID REFERENCES materials(id) ON DELETE CASCADE,
    chemical_name VARCHAR(255) NOT NULL,
    hazards JSONB NOT NULL,
    handling TEXT NOT NULL,
    storage TEXT NOT NULL,
    first_aid TEXT NOT NULL,
    disposal TEXT NOT NULL,
    file_link VARCHAR(255),
    last_updated TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### 5. Recipes

- **Purpose**: Core entity for recipes managed via conversational commands, subject to standard enforcement and safety checks.
- **Attributes**:
  - `id`: Unique identifier.
  - `workspace_id`: Link to Workspace (data isolation).
  - `name`: Recipe name.
  - `description`: Brief overview.
  - `ingredients`: Materials and quantities (JSON).
  - `steps`: Procedure (array of text).
  - `conditions`: Environmental factors (JSON).
  - `yield`: Efficiency percentage.
  - `cost`: Calculated cost.
  - `creator_id`: Link to Profile.
  - `created_at`: Creation timestamp.
  - `updated_at`: Last update timestamp.
  - `status`: Workflow stage (e.g., "Draft").
  - `version`: Version number for memory.
  - `generated_sds_id`: Link to AI-generated SDS.
- **DDL**:

```sql
CREATE TABLE recipes (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id UUID REFERENCES workspaces(id) NOT NULL,
    name VARCHAR(255) NOT NULL,
    description TEXT,
    ingredients JSONB NOT NULL,
    steps TEXT[] NOT NULL,
    conditions JSONB,
    yield DECIMAL,
    cost DECIMAL,
    creator_id UUID REFERENCES profiles(id) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    status VARCHAR(50) CHECK (status IN ('Draft', 'Tested', 'Ready to Sell')) DEFAULT 'Draft',
    version INT DEFAULT 1,
    generated_sds_id UUID REFERENCES generated_sds(id)
);
```

### 6. Experiments (Recipe Tests)

- **Purpose**: Validate recipes, capture structured test results, and provide data for the agent's knowledge retrieval about past performance and issues.
- **Attributes**:
  - `id`: Unique identifier.
  - `workspace_id`: Link to Workspace (data isolation).
  - `recipe_id`: Link to Recipe.
  - `date`: Test date.
  - `researcher_id`: Link to Profile.
  - `materials_used`: Materials consumed (JSON).
  - `results`: Outcome (e.g., yield).
  - `notes`: Observations.
- **DDL**:

```sql
CREATE TABLE experiments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id UUID REFERENCES workspaces(id) NOT NULL,
    recipe_id UUID REFERENCES recipes(id) ON DELETE CASCADE,
    date DATE NOT NULL,
    researcher_id UUID REFERENCES profiles(id) NOT NULL,
    materials_used JSONB NOT NULL,
    results JSONB,
    notes TEXT
);
```

### 7. Generated SDS

- **Purpose**: Store AI-generated safety data for recipes, managed and reviewed via the conversational agent, contributing to safety checks.
- **Attributes**:
  - `id`: Unique identifier.
  - `workspace_id`: Link to Workspace (data isolation).
  - `recipe_id`: Link to Recipe.
  - `generated_name`: Recipe-specific name (e.g., "Batch 001").
  - `hazards`: Key safety risks (JSON).
  - `handling`: Precautions.
  - `storage`: Storage needs.
  - `first_aid`: Emergency steps.
  - `disposal`: Disposal instructions.
  - `file_link`: Generated document URL.
  - `confidence_score`: AI reliability (0-1).
  - `status`: "Draft," "Reviewed," "Final."
  - `created_at`: Generation timestamp.
- **DDL**:

```sql
CREATE TABLE generated_sds (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id UUID REFERENCES workspaces(id) NOT NULL,
    recipe_id UUID REFERENCES recipes(id) ON DELETE CASCADE,
    generated_name VARCHAR(255) NOT NULL,
    hazards JSONB NOT NULL,
    handling TEXT NOT NULL,
    storage TEXT NOT NULL,
    first_aid TEXT NOT NULL,
    disposal TEXT NOT NULL,
    file_link VARCHAR(255),
    confidence_score DECIMAL CHECK (confidence_score BETWEEN 0 AND 1),
    status VARCHAR(50) CHECK (status IN ('Draft', 'Reviewed', 'Final')) DEFAULT 'Draft',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### 8. Conversations (Agent Interaction Log)

- **Purpose**: Log conversational turns for auditing, debugging, agent improvement, and as a source for advanced knowledge retrieval about past discussions and decisions.
- **Attributes**:
  - `id`: Unique identifier for the conversation turn.
  - `workspace_id`: Link to Workspace (data isolation).
  - `session_id`: Identifier linking turns within a single conversation session.
  - `user_id`: Link to auth.users via Profile.
  - `user_input`: The text entered by the user.
  - `agent_response`: The text response generated by the agent.
  - `intent_detected`: The intent identified by the NLU (e.g., `create_recipe`).
  - `entities_extracted`: Entities extracted by the NLU (JSON, e.g., `{"recipe_name": "Aspirin"}`).
  - `action_taken`: Backend action orchestrated by the agent (e.g., `RecipeService.create`).
  - `timestamp`: Timestamp of the interaction turn.
- **DDL**:

```sql
CREATE TABLE conversations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id UUID REFERENCES workspaces(id) NOT NULL,
    session_id UUID NOT NULL,
    user_id UUID REFERENCES auth.users(id),
    user_input TEXT,
    agent_response TEXT,
    intent_detected VARCHAR(100),
    entities_extracted JSONB,
    action_taken VARCHAR(255),
    timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
-- Indexes for faster queries
CREATE INDEX idx_conversations_session_id ON conversations(session_id);
CREATE INDEX idx_conversations_workspace_id ON conversations(workspace_id);
CREATE INDEX idx_conversations_user_id ON conversations(user_id);
```

### 9. Standard Rules

- **Purpose**: Store configurable standards and validation rules enforced by the AI agent, scoped to workspaces.
- **Attributes**:
  - `id`: Unique identifier.
  - `workspace_id`: Link to Workspace (data isolation).
  - `rule_name`: Descriptive name (e.g., "ConcentrationFormat", "MandatorySafetyNote").
  - `rule_type`: Type of rule (e.g., "Regex", "PresenceCheck", "Compatibility").
  - `configuration`: Rule details (JSON, e.g., `{"pattern": "\\d+M"}`, `{"field": "steps.safety_note"}`).
  - `error_message`: Message for the agent to use when rule is violated.
  - `is_active`: Boolean flag.
- **DDL**:

```sql
CREATE TABLE standard_rules (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id UUID REFERENCES workspaces(id) NOT NULL,
    rule_name VARCHAR(100) NOT NULL,
    rule_type VARCHAR(50) NOT NULL,
    configuration JSONB,
    error_message TEXT,
    is_active BOOLEAN DEFAULT true,
    UNIQUE(workspace_id, rule_name)
);
```

### 10. Audit Log

- **Purpose**: Explicitly log critical actions and decisions for compliance and traceability, supplementing the general `Conversations` log.
- **Attributes**:
  - `id`: Unique identifier.
  - `workspace_id`: Link to Workspace (data isolation).
  - `timestamp`: Time of the event.
  - `user_id`: User performing the action (via auth.users).
  - `action_type`: Type of critical action (e.g., "SDS_Approve", "Recipe_Status_Change", "Safety_Warning_Acknowledge").
  - `entity_id`: ID of the affected entity (e.g., recipe ID, SDS ID).
  - `details`: Contextual details (JSON, e.g., `{"old_status": "Draft", "new_status": "Ready to Sell"}`).
  - `conversation_turn_id`: Link to the relevant turn in the `Conversations` table (optional).
- **DDL**:

```sql
CREATE TABLE audit_log (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id UUID REFERENCES workspaces(id) NOT NULL,
    timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    user_id UUID REFERENCES auth.users(id),
    action_type VARCHAR(100) NOT NULL,
    entity_id UUID,
    details JSONB,
    conversation_turn_id UUID REFERENCES conversations(id)
);
CREATE INDEX idx_audit_log_workspace_id ON audit_log(workspace_id);
CREATE INDEX idx_audit_log_user_id ON audit_log(user_id);
```

## Relationships

- **Profiles ↔ Auth Users**: One-to-one (each profile links to exactly one Supabase auth user).
- **Profiles ↔ Assigned Workspace**: Many-to-one (each profile is assigned to exactly one workspace by developers).
- **Workspaces ↔ All Data Entities**: One-to-many (each workspace contains its own isolated set of recipes, materials, SDS, conversations, etc.).
- **Profiles ↔ Recipes**: One-to-many (one profile creates many recipes), constrained by workspace.
- **Materials ↔ Material SDS**: One-to-one (each material has one SDS), constrained by workspace.
- **Recipes ↔ Materials**: Many-to-many via `ingredients` JSON, constrained by workspace.
- **Recipes ↔ Experiments**: One-to-many (one recipe has many tests), constrained by workspace.
- **Recipes ↔ Generated SDS**: One-to-one (each recipe gets one SDS), constrained by workspace.
- **Workspace ↔ Standard Rules**: One-to-many (each workspace defines its own validation rules).
- **Auth Users ↔ Conversations**: One-to-many (one user has many conversation turns), constrained by workspace.
- **Audit Log**: Links to Auth Users, Conversations, and conceptually to other entities via `entity_id`; all constrained by workspace.

## How Entities Support Goals

- **Workspace Isolation**: All data is partitioned by `workspace_id`, ensuring teams work in their own secure environments with private data.
- **Workspace-Specific AI Behavior**: Each workspace's `custom_instructions` guides the AI agent's responses and capabilities, allowing for tailored experiences across different teams.
- **Agent Configuration Flexibility**: The `agent_configurations` field enables precise control over each agent's behavior, model selection, tool availability, and specialized instructions within each workspace.
- **Conversational Recipe Management**: `Recipes`, `Materials`, `Profiles` are manipulated by the AI Agent, guided by workspace-specific `Standard Rules` and logged in `Conversations` and `Audit Log`.
- **Conversational Education & Safety**: `Material SDS`, `Generated SDS`, and safety data are used by the agent for guidance and proactive checks within the workspace context.
- **Organizational Memory & Knowledge Retrieval**: `Recipes` (versioned), `Experiments`, and `Conversations` provide the raw data for knowledge retrieval, scoped to the workspace for relevance and security.

## Supabase Integration

This schema is designed to work with Supabase, where:

- Authentication is handled by Supabase Auth (auth schema)
- `profiles` table extends the built-in `auth.users` table with application-specific data
- Row Level Security (RLS) policies are applied to all tables to enforce workspace isolation
- Auth hooks automatically create profile records when new users are registered

## Integration with Conversational AI

The AI agent operates within the context of the user's assigned workspace:

1. **Workspace-Specific Instructions**

   - AI agent reads `custom_instructions` from the workspace table at the start of each conversation
   - These instructions guide the agent's tone, style, safety protocols, and domain-specific behaviors
   - All responses are tailored according to these workspace-level instructions

2. **Agent Configuration**

   - System reads `agent_configurations` from the workspace table
   - Different agents are dynamically created based on these configurations
   - Each agent type (Workspace, Recipe, Safety, Knowledge) can have custom settings:
     - Model selection (e.g., GPT-4 for critical agents)
     - Temperature settings for response variability
     - Enabled tools specific to each agent
     - Specialized instructions beyond the general workspace instructions

3. **Session Context**
   - AI agent receives user's JWT from Supabase with each request
   - JWT contains assigned workspace context
   - All data retrieval and mutations respect workspace boundaries through RLS

## Future Enhancements (Post-MVP)

For future releases, the database structure will be extended to support:

1. **Self-Service Workspace Creation**

   - Adding owner relationship to workspaces
   - Workspace customization options
   - UI for editing custom instructions
   - Self-service agent configuration interface

2. **Workspace Membership**

   - Implementation of `workspace_members` table
   - Role-based permissions within workspaces

3. **Role Management**

   - Enhanced user roles (Admin, Member)
   - Chemistry-specific roles (Newcomer, Experienced)
   - Role-based access to agent configuration

4. **Multi-Workspace Support**
   - Allowing users to belong to multiple workspaces
   - Supporting workspace switching via `current_workspace_id`
   - Workspace-specific agent configuration templates
