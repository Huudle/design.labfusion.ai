# Safety Data Sheet (SDS) System (via Agent)

## Overview

The AI Agent interacts with the SDS system components to:

1.  Ensure safety compliance for recipes, including proactive checks.
2.  Educate users about safety protocols and standards through conversation.
3.  Generate and manage SDS documents based on user requests and safety analysis.
4.  Leverage SDS data for proactive hazard warnings and knowledge retrieval.

## Core Components (Accessed by AI Agent)

### 1. Material SDS Database

- Stores SDS information for chemicals.
- **Agent Interaction**: Queries this database via the SDS Generation Service for specific material hazards, safe handling, and disposal information. This data feeds into **proactive safety checks** during recipe discussion.

### 2. AI-Powered SDS Generation Service

- Generates recipe SDS based on ingredients/conditions.
- **Agent Interaction**: Agent triggers this service for recipe SDS generation. The agent also uses the underlying logic (or data from this service) to perform **proactive safety checks** regarding ingredient compatibility or process risks.

### 3. Safety Service

- Stores general safety guidelines, emergency contacts, risk indicators, and potentially company-specific safety rules/standards.
- **Agent Interaction**: Agent queries this service for general safety info and uses its rules/data to contribute to **proactive safety checks** and **standard enforcement** during recipe creation/modification.

### 4. Version Control

- Tracks SDS versions.
- **Agent Interaction**: The agent ensures that SDS generation and approval actions are properly versioned through the backend services.

## User Roles (Enforced by AI Agent)

### Newcomers

- Can ask the agent to display SDS for materials and recipes.
- Can ask the agent basic safety questions.
- Cannot ask the agent to finalize/approve SDS.

### Experienced Chemists

- Can ask the agent to display, generate, edit (if supported), and approve SDS.
- Can ask the agent to update basic safety information (if supported by Safety Service API).

## Integration (Orchestrated by AI Agent)

- **Recipe Management**: Agent links SDS requests/generation/checks to specific recipes, enforces safety/standard checkpoints during recipe lifecycle.
- **Inventory**: Agent uses material information when retrieving Material SDS.
- **Safety Service**: Agent retrieves safety rules, guidelines, and risk levels for proactive checks and standard enforcement.
- **Knowledge Retrieval**: SDS data (material and generated) can be included in responses to complex safety queries.

## Security and Compliance (Managed via Agent)

- **Role-Based Access**: Agent checks user role before executing SDS-related actions (e.g., approval).
- **Audit Logging**: Agent interactions, especially those involving safety warnings, acknowledgements, and SDS approvals, are logged for auditability.
- **Compliance**: Agent relies on accurate Material SDS, robust proactive check logic, standard enforcement rules, and Experienced Chemist reviews for compliance.
