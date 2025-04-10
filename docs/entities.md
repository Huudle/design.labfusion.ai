# Database Entities

## Overview

This document outlines the database structure and relationships for the labfusion.ai conversational AI system. Each entity is designed to support conversational recipe management, proactive safety, standard enforcement, education, and organizational memory access.

## Entities

### 1. Users

- **Purpose**: Manage access, track creators/editors, and support role-based permissions within the conversational agent.
- **Attributes**:
  - `id`: Unique identifier.
  - `name`: User's full name.
  - `role`: "Newcomer" or "Experienced" for permissions enforced by the AI agent.
  - `email`: Login credential.
  - `created_at`: Timestamp.
- **DDL**:

```sql
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    role VARCHAR(50) CHECK (role IN ('Newcomer', 'Experienced')) NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### 2. Materials (Inventory)

- **Purpose**: Track materials for recipes, pricing, and safety data.
- **Attributes**:
  - `id`: Unique identifier.
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

### 3. Material SDS

- **Purpose**: Store pre-existing safety data for materials, used by the agent for proactive checks, education, and AI-generated SDS.
- **Attributes**:
  - `id`: Unique identifier.
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

### 4. Recipes

- **Purpose**: Core entity for recipes managed via conversational commands, subject to standard enforcement and safety checks.
- **Attributes**:
  - `id`: Unique identifier.
  - `name`: Recipe name.
  - `description`: Brief overview.
  - `ingredients`: Materials and quantities (JSON).
  - `steps`: Procedure (array of text).
  - `conditions`: Environmental factors (JSON).
  - `yield`: Efficiency percentage.
  - `cost`: Calculated cost.
  - `creator_id`: Link to User.
  - `created_at`: Creation timestamp.
  - `updated_at`: Last update timestamp.
  - `status`: Workflow stage (e.g., "Draft").
  - `version`: Version number for memory.
  - `generated_sds_id`: Link to AI-generated SDS.
- **DDL**:

```sql
CREATE TABLE recipes (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    description TEXT,
    ingredients JSONB NOT NULL,
    steps TEXT[] NOT NULL,
    conditions JSONB,
    yield DECIMAL,
    cost DECIMAL,
    creator_id UUID REFERENCES users(id) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    status VARCHAR(50) CHECK (status IN ('Draft', 'Tested', 'Ready to Sell')) DEFAULT 'Draft',
    version INT DEFAULT 1,
    generated_sds_id UUID REFERENCES generated_sds(id)
);
```

### 5. Experiments (Recipe Tests)

- **Purpose**: Validate recipes, capture structured test results, and provide data for the agent's knowledge retrieval about past performance and issues.
- **Attributes**:
  - `id`: Unique identifier.
  - `recipe_id`: Link to Recipe.
  - `date`: Test date.
  - `researcher_id`: Link to User.
  - `materials_used`: Materials consumed (JSON).
  - `results`: Outcome (e.g., yield).
  - `notes`: Observations.
- **DDL**:

```sql
CREATE TABLE experiments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    recipe_id UUID REFERENCES recipes(id) ON DELETE CASCADE,
    date DATE NOT NULL,
    researcher_id UUID REFERENCES users(id) NOT NULL,
    materials_used JSONB NOT NULL,
    results JSONB,
    notes TEXT
);
```

### 6. Generated SDS

- **Purpose**: Store AI-generated safety data for recipes, managed and reviewed via the conversational agent, contributing to safety checks.
- **Attributes**:
  - `id`: Unique identifier.
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

### 7. Conversations (Agent Interaction Log)

- **Purpose**: Log conversational turns for auditing, debugging, agent improvement, and as a source for advanced knowledge retrieval about past discussions and decisions.
- **Attributes**:
  - `id`: Unique identifier for the conversation turn.
  - `session_id`: Identifier linking turns within a single conversation session.
  - `user_id`: Link to User.
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
    session_id UUID NOT NULL, -- Or potentially TEXT depending on session ID generation
    user_id UUID REFERENCES users(id),
    user_input TEXT,
    agent_response TEXT,
    intent_detected VARCHAR(100),
    entities_extracted JSONB,
    action_taken VARCHAR(255),
    timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
-- Optional index for querying sessions
CREATE INDEX idx_conversations_session_id ON conversations(session_id);
```

### 8. (Potential) Standard Rules

- **Purpose**: Store configurable company standards and validation rules enforced by the AI agent.
- **Attributes**:
  - `id`: Unique identifier.
  - `rule_name`: Descriptive name (e.g., "ConcentrationFormat", "MandatorySafetyNote").
  - `rule_type`: Type of rule (e.g., "Regex", "PresenceCheck", "Compatibility").
  - `configuration`: Rule details (JSON, e.g., `{"pattern": "\\d+M"}`, `{"field": "steps.safety_note"}`).
  - `error_message`: Message for the agent to use when rule is violated.
  - `is_active`: Boolean flag.
- **DDL (Example)**:

```sql
CREATE TABLE standard_rules (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    rule_name VARCHAR(100) UNIQUE NOT NULL,
    rule_type VARCHAR(50) NOT NULL,
    configuration JSONB,
    error_message TEXT,
    is_active BOOLEAN DEFAULT true
);
```

### 9. (Potential) Audit Log

- **Purpose**: Explicitly log critical actions and decisions for compliance and traceability, supplementing the general `Conversations` log.
- **Attributes**:
  - `id`: Unique identifier.
  - `timestamp`: Time of the event.
  - `user_id`: User performing the action (via agent).
  - `action_type`: Type of critical action (e.g., "SDS_Approve", "Recipe_Status_Change", "Safety_Warning_Acknowledge").
  - `entity_id`: ID of the affected entity (e.g., recipe ID, SDS ID).
  - `details`: Contextual details (JSON, e.g., `{"old_status": "Draft", "new_status": "Ready to Sell"}`).
  - `conversation_turn_id`: Link to the relevant turn in the `Conversations` table (optional).
- **DDL (Example)**:

```sql
CREATE TABLE audit_log (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    user_id UUID REFERENCES users(id),
    action_type VARCHAR(100) NOT NULL,
    entity_id UUID,
    details JSONB,
    conversation_turn_id UUID REFERENCES conversations(id) -- Optional link
);
```

## Relationships

- **Users ↔ Recipes**: One-to-many (one user creates many recipes).
- **Materials ↔ Material SDS**: One-to-one (each material has one SDS).
- **Recipes ↔ Materials**: Many-to-many via `ingredients` JSON.
- **Recipes ↔ Experiments**: One-to-many (one recipe has many tests).
- **Recipes ↔ Generated SDS**: One-to-one (each recipe gets one SDS).
- **Users ↔ Conversations**: One-to-many (one user has many conversation turns).
- **(Potential)** Standard Rules apply conceptually during agent validation.
- **(Potential)** Audit Log links to Users, optionally Conversations, and conceptually to other entities via `entity_id`.

## How Entities Support Goals

- **Conversational Recipe Management**: `Recipes`, `Materials`, `Users` are manipulated by the AI Agent, guided by `Standard Rules` and logged in `Conversations` and `Audit Log`.
- **Conversational Education & Safety**: `Material SDS`, `Generated SDS`, and `Safety Service` data are used by the agent for guidance and proactive checks. `Conversations` track guidance.
- **Organizational Memory & Knowledge Retrieval**: `Recipes` (versioned), `Experiments`, and especially `Conversations` provide the raw data. The agent (potentially via `Knowledge Service`) synthesizes insights from these tables.
