# Architecture

## Architectural Style

**Conversational AI-Driven Architecture** with **Backend Microservices**:

- **Conversational AI**: Primary user interaction via chat, leveraging NLU, dialog management, standard enforcement, proactive safety checks, and knowledge retrieval.
- **Microservices**: Backend logic encapsulated in distinct services, orchestrated by the AI agent.
- **Serverless**: Backend services and AI components deployed using serverless functions.

## System Architecture

```
[Client Layer]
   Chat Interface (Web App - Next.js)
   └── Minimal GUI for Visualization (Optional)

[API Gateway]
   RESTful API (Primarily for /chat endpoint)

[Service Layer]
   ├── AI Agent Service (NLU, Dialog, Orchestration, Safety Checks, Standards, Knowledge Retrieval) <--- CENTRAL
   │    └── Calls other services -->
   ├── Recipe Service (incl. Validation Logic?)
   ├── Inventory Service
   ├── SDS Generation Service
   ├── Safety Service
   ├── User Service
   └── (Maybe) Knowledge Service (Handles complex queries across tables?)

[Data Layer]
   ├── PostgreSQL Database (Supabase)
   │    ├── Tables: users, materials, material_sds, recipes, generated_sds, experiments, conversations, standards_rules?, audit_log?
   │    └── Storage: SDS PDFs
   └── External Data / AI Models (LLM, NLU models, Vector DB?)
```

## Components & Responsibilities

### 1. Client Layer (Front-End)

- **Tech**: Next.js
- **Primary Component**: **Chat Interface**
  - Accepts user text input (commands, questions).
  - Sends messages to the API Gateway.
  - Displays responses from the AI Agent Service (text, potentially cards or links).
- **Optional Component**: Minimal GUI for visualizing complex data (e.g., displaying a full recipe table or SDS document presented by the agent).

### 2. API Gateway

- **Tech**: Next.js API Routes or dedicated gateway.
- **Primary Role**: Routes incoming chat messages (e.g., from `/api/chat`) to the central AI Agent Service.
- Handles authentication (via User Service/Supabase Auth).

### 3. Service Layer

#### AI Agent Service (Central Orchestrator)

- **Purpose**: Understands user requests, manages dialog, enforces standards, performs proactive safety checks, retrieves knowledge, and coordinates actions.
- **Responsibilities**:
  - **Natural Language Understanding (NLU)**: Interprets user text to identify intents and extract entities, including complex queries requiring synthesis.
  - **Dialog Management**: Manages conversation flow, asks clarifying questions if needed, maintains context.
  - **Standard Enforcement**: Validates user inputs/recipe details against predefined rules (potentially fetched from DB or config). Rejects or prompts for corrections.
  - **Proactive Safety Analysis**: Analyzes recipe ingredients/steps during interaction, cross-references SDS/historical data, and issues warnings.
  - **Knowledge Retrieval**: Queries `Recipes`, `Experiments`, `Conversations`, and `SDS` data (potentially via a dedicated Knowledge Service or direct DB access) to answer complex user questions.
  - **Orchestration**: Determines which backend service(s) to call based on the user's intent and extracted entities.
  - **Response Generation**: Formulates responses, including warnings, standard guidance, and synthesized knowledge.
- **Technology**: Requires integration with LLMs, NLU frameworks, potentially rule engines, and agent frameworks (like LangChain). May need access to vector databases for semantic search over logs/docs.

#### Recipe Service

- **Purpose**: Manages `Recipes` data, potentially including validation logic.
- **Responsibilities**: CRUD operations, cost/yield, versioning. May contain specific validation logic related to recipe structure, triggered by the AI Agent.
- **Interaction**: Primarily serves requests from the AI Agent Service (e.g., "Create this recipe", "Retrieve recipe X", "Update ingredient list for recipe Y").

#### Inventory Service

- **Purpose**: Manages `Materials` data.
- **Responsibilities**: Stock level tracking, basic alerts.
- **Interaction**: Primarily serves requests from the AI Agent Service (e.g., "Check stock for material Z", "Reduce stock for material A used in recipe B").

#### SDS Generation Service

- **Purpose**: Manages `Material SDS` and `Generated SDS`.
- **Responsibilities**: Stores Material SDS, triggers AI-based generation of recipe SDS, manages PDF storage.
- **Interaction**: Primarily serves requests from the AI Agent Service (e.g., "Generate SDS for recipe C", "Retrieve Material SDS for material D").

#### Safety Service

- **Purpose**: Provides core safety information and risk data.
- **Responsibilities**: Stores emergency contacts, guidelines, risk indicators. Provides data for the AI Agent's proactive checks.
- **Interaction**: Primarily serves requests from the AI Agent Service (e.g., "What are the safety guidelines?", "Retrieve emergency contacts").

#### User Service

- **Purpose**: Manages `Users`, authentication, and potentially detailed auditing.
- **Responsibilities**: User profiles, roles, permissions. May manage or receive detailed audit logs for critical actions triggered via the agent.
- **Interaction**: Serves requests from the API Gateway (authentication) and potentially the AI Agent Service (checking permissions for actions).

#### (Potential) Knowledge Service

- **Purpose**: Dedicated service to handle complex queries requiring joins/analysis across multiple tables (`Recipes`, `Experiments`, `Conversations`).
- **Responsibilities**: Executes complex search/aggregation queries, potentially using vector search or other AI techniques.
- **Interaction**: Serves requests from the AI Agent Service for Advanced Knowledge Retrieval.

### 4. Data Layer

- **Tech**: Supabase (PostgreSQL + Storage), potentially Vector Database.
- **Database**: Stores core entities. `Conversations` and `Experiments` tables are crucial for organizational memory and knowledge retrieval. May need tables for `standards_rules` and a dedicated `audit_log` for critical actions.
- **Storage**: SDS PDFs.
- **External Data / AI Models**: Relies on external LLMs, NLU models, or hosted AI services for the AI Agent Service's core functionality.

## Data Flow Example (Conversational Recipe Creation with Checks)

1.  **Client**: User types "Add 50ml of concentrated Hydrochloric Acid to the 'Test Reaction' recipe."
2.  **API Gateway**: Routes message to AI Agent Service.
3.  **AI Agent Service**:
    - NLU identifies intent: `update_recipe_ingredient`, extracts entities...
    - **Standard Enforcement**: Checks if concentration specified meets standards (e.g., requires molarity). _Agent Responds_: "Please specify the concentration for Hydrochloric Acid as per company standard (e.g., 12M)."
    - _User_: "Okay, add 50ml of 12M Hydrochloric Acid."
    - **Proactive Safety Check**: Agent analyzes existing ingredients in 'Test Reaction' and checks compatibility with HCl based on SDS/rules. _Agent Responds_: "Warning: Adding concentrated HCl to the existing base might cause a strong reaction. Ensure proper ventilation and cooling. Do you want to proceed?"
    - _User_: "Yes, proceed."
    - **Orchestration**: Agent calls Recipe Service API to update the ingredient.
    - **(Auditing)**: Agent logs the critical addition (with warning acknowledgement) potentially via User Service or direct log.
    - **Response Generation**: "Okay, I've added 50ml of 12M Hydrochloric Acid to the 'Test Reaction' recipe. Please remember the safety warning."
4.  **API Gateway**: Forwards response to Client.
5.  **Client**: Displays agent's response.

## Non-Functional Considerations

- **Scalability**: AI Agent Service, particularly knowledge retrieval and analysis components, needs careful scaling.
- **Performance**: NLU, complex query synthesis, proactive checks add latency; needs optimization for conversational flow.
- **Reliability**: Crucial for NLU accuracy, standard enforcement logic, safety check rules, and knowledge retrieval correctness. Confirmation steps are vital.
- **Security**: Role-based access enforced by agent; protection of sensitive data in logs (`Conversations`).
- **Maintainability**: Clear APIs; potentially complex logic within AI Agent Service needs good structure. Rule engine for standards might help.
- **Safety**: Robustness of proactive checks is critical. Agent must not give incorrect safety advice or bypass checks.
- **Auditability**: Comprehensive logging in `Conversations` / `audit_log` is essential.
