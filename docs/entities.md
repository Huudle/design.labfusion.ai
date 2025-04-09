# Database Entities

## Overview

This document outlines the database structure and relationships for the labfusion.ai system. Each entity is designed to support the core goals of recipe management, education, and organizational memory.

## Entities

### 1. Users

- **Purpose**: Manage access, track creators/editors, and support role-based education.
- **Attributes**:
  - `id`: Unique identifier.
  - `name`: User's full name.
  - `role`: "Novice" or "Experienced" for permissions.
  - `email`: Login credential.
  - `created_at`: Timestamp for organizational memory.
- **DDL**:

```sql
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    role VARCHAR(50) CHECK (role IN ('Novice', 'Experienced')) NOT NULL,
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

- **Purpose**: Store pre-existing safety data for materials, used for education and AI-generated SDS.
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

- **Purpose**: Core entity for creating sellable recipes and organizational memory.
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

- **Purpose**: Validate recipes, educate novices, and build memory.
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

- **Purpose**: AI-generated safety data for recipes, editable, and educational.
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

### 7. Customer Summaries

- **Purpose**: Enhance sellability with customer-facing info.
- **Attributes**:
  - `id`: Unique identifier.
  - `recipe_id`: Link to Recipe.
  - `description`: Sales-friendly overview.
  - `benefits`: Key selling points (e.g., "Low cost").
  - `sds_highlights`: Simplified safety info.
  - `file_link`: Exportable summary URL.
  - `created_at`: Creation timestamp.
- **DDL**:

```sql
CREATE TABLE customer_summaries (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    recipe_id UUID REFERENCES recipes(id) ON DELETE CASCADE,
    description TEXT NOT NULL,
    benefits JSONB NOT NULL,
    sds_highlights TEXT NOT NULL,
    file_link VARCHAR(255),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### 8. Queries (AI Interaction Log)

- **Purpose**: Log AI queries for education and memory.
- **Attributes**:
  - `id`: Unique identifier.
  - `user_id`: Link to User.
  - `question`: User's query.
  - `response`: AI answer.
  - `created_at`: Timestamp.
- **DDL**:

```sql
CREATE TABLE queries (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id) NOT NULL,
    question TEXT NOT NULL,
    response TEXT NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

## Relationships

- **Users ↔ Recipes**: One-to-many (one user creates many recipes).
- **Materials ↔ Material SDS**: One-to-one (each material has one SDS).
- **Recipes ↔ Materials**: Many-to-many via `ingredients` JSON.
- **Recipes ↔ Experiments**: One-to-many (one recipe has many tests).
- **Recipes ↔ Generated SDS**: One-to-one (each recipe gets one SDS).
- **Recipes ↔ Customer Summaries**: One-to-one (each sellable recipe has one summary).
- **Users ↔ Queries**: One-to-many (one user asks many questions).

## How Entities Support Goals

- **Selling Recipes**: `Recipes`, `Generated SDS`, and `Customer Summaries` provide a complete package (recipe + safety + sales pitch).
- **Educating Novices**:
  - `Material SDS` and `Generated SDS` teach safety with accessible data.
  - `Recipes` and `Experiments` store design examples; JSON fields allow detailed learning.
- **Organizational Memory**: Versioning (`Recipes.version`), timestamps, and `Queries` preserve history.
