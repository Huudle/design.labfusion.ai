# Requirements

## Core Purpose

The conversational AI system will:

1. **Enable chemists to create, manage, and refine chemical recipes through natural language interaction**, enforcing safety and standards (primary goal).
2. **Educate newcomers** on safety and recipe design via conversational guidance and access to organizational knowledge.
3. **Create and provide access to an organizational memory** by capturing and synthesizing interactions, recipe data, and experimental results.

## Functional Requirements

### 1. Conversational AI Agent

- **Purpose**: Understand user requests, manage dialog, enforce standards, provide proactive guidance, and orchestrate actions.
- **Requirements**:
  - **Natural Language Understanding (NLU)**: Accurately interpret user text input to identify intents (e.g., `create_recipe`, `query_sds`, `check_inventory`) and extract relevant entities (e.g., recipe names, material names, quantities, safety terms).
  - **Dialog Management**: Maintain conversation context, ask clarifying questions for ambiguous requests, handle multi-turn interactions.
  - **Action Orchestration**: Trigger appropriate backend service APIs (Recipe, Inventory, SDS, Safety) based on understood intent and entities.
  - **Response Generation**: Provide clear, concise, and relevant natural language responses to the user.
  - **Confirmation**: For critical actions (e.g., finalizing SDS, changing recipe status), confirm user intent before proceeding.
  - **Proactive Safety Checks**: During recipe creation/editing, analyze ingredients and steps to proactively warn users about known hazardous combinations or process risks based on `Material SDS` and potentially historical `Conversations`/`Experiments` data (e.g., "Warning: Mixing substance A and B can be exothermic.").
  - **Standard Enforcement**: Validate user inputs and recipe definitions against predefined company standards and rules (e.g., format requirements, mandatory fields, incompatible materials). Reject or prompt for correction if standards are not met.
  - **Advanced Knowledge Retrieval**: Answer complex queries by synthesizing information across `Recipes`, `Experiments`, and `Conversations` logs (e.g., "What were the common issues encountered when testing recipes similar to X?", "Summarize safety discussions related to substance Y last quarter.").

### 2. Recipe Management (via Agent)

- **Purpose**: Allow users to manage recipes through conversation while adhering to standards.
- **Requirements**:
  - Agent can create, edit, retrieve, and list recipes based on user commands (e.g., "Create a recipe...", "Add X to recipe Y", "Show me recipe Z", "List my draft recipes").
  - Agent understands recipe attributes (Name, Ingredients, Steps, Conditions, etc.) mentioned in user requests.
  - **Validation**: Agent enforces validation rules during creation/editing (e.g., checks for required fields, valid units, adherence to company standards defined in configuration).
  - **(Optional) Template Usage**: Agent may suggest or require the use of predefined recipe templates for certain processes to ensure standardization.
  - Agent enforces versioning when recipes are modified.
  - Agent provides recipe cost/yield information upon request.

### 3. Inventory Management (via Agent)

- **Purpose**: Allow users to query inventory through conversation.
- **Requirements**:
  - Agent can report material details (Quantity, Location, Price) based on user queries (e.g., "How much water do we have?", "What's the price of acetone?").
  - Agent informs backend services to update inventory when materials are used in recipes.

### 4. Safety & SDS Management (via Agent)

- **Purpose**: Provide safety information, manage SDS, and perform proactive checks via conversation.
- **Requirements**:
  - Agent can retrieve and display Material SDS information (e.g., "Show SDS for ethanol").
  - Agent can trigger the generation of a recipe's SDS (e.g., "Generate SDS for recipe X").
  - Agent allows Experienced Chemists to review and approve generated SDS via conversation (with appropriate confirmations).
  - Agent can answer basic safety questions by querying the Safety Service (e.g., "What are the emergency contacts?").
  - Agent presents risk level indicators when discussing recipes.
  - **Proactive Hazard Identification**: Reinforces the agent's duty to identify and communicate potential hazards during recipe discussion, based on ingredient properties and process steps.

### 5. User Management & Permissions (via Agent)

- **Purpose**: Control access and maintain audit trails through the agent.
- **Requirements**:
  - Agent identifies the user and their role (Newcomer, Experienced).
  - Agent restricts actions based on permissions (e.g., prevents Newcomers from finalizing SDS or editing others' recipes unless in draft).
  - Authentication handled primarily at API Gateway / User Service level.
  - **Detailed Auditing**: Key actions orchestrated by the agent (e.g., recipe status changes, SDS approval, critical edits) must be logged with user, timestamp, and context (potentially within the `Conversations` table or a dedicated audit log) for traceability.

### 6. Conversational Education & Guidance

- **Purpose**: Guide newcomers and enforce standards through conversation.
- **Requirements**:
  - Agent provides contextual help and tips during recipe creation/editing for Newcomers.
  - Agent proactively offers safety information relevant to the conversation.
  - Agent can answer basic procedural questions (e.g., "How do I add an ingredient?").
  - **Standard Guidance**: Agent explains company standards or validation rules when prompting users for corrections (e.g., "Please specify temperature in Celsius as per company standard.").

## Non-Functional Requirements

- **Usability**: Conversational interface should be intuitive; agent must handle variations in user phrasing.
- **Performance**: AI Agent response time (including NLU, orchestration, response generation, and knowledge retrieval) should be reasonably fast for a fluid conversation.
- **Reliability**: High accuracy in NLU, action mapping, standard enforcement, and knowledge retrieval. Graceful handling of misunderstood requests. Robust confirmation for critical actions.
- **Security**: Role-based permissions enforced by the agent before executing actions. Secure handling of user data.
- **Safety**: Agent design prioritizes safety, avoiding misinterpretation of safety-critical commands. Agent must surface safety warnings appropriately and reliably perform proactive safety checks.
- **Scalability**: AI Agent Service and backend microservices must scale to handle concurrent conversations.
- **Accessibility**: Chat interface should adhere to web accessibility standards.
- **Auditability**: Logs (e.g., `Conversations`) must be comprehensive enough to reconstruct critical decision paths and standard adherence.

## Database Implementation

### Conversations Table

The `conversations` table stores the chat interactions between users and the AI agent:

```sql
CREATE TABLE public.conversations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id UUID REFERENCES public.workspaces(id) NOT NULL,
    messages JSONB NOT NULL DEFAULT '[]'::jsonb,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT now(),
    profile_id UUID NOT NULL REFERENCES public.profiles(id),
    is_canvas_created BOOLEAN NOT NULL DEFAULT false,
    agent TEXT NOT NULL DEFAULT 'RECIPE_GENERATOR'::text,
    title TEXT NULL,
    canvas_content TEXT NULL
);
```

- **Purpose**: Store complete conversation threads between users and the AI agent
- **Key Fields**:

  - `id`: Unique identifier for the conversation
  - `workspace_id`: Ensures data isolation between workspaces
  - `messages`: JSONB array containing the entire conversation history
  - `profile_id`: Links to the user's profile
  - `agent`: Type of AI agent used (e.g., 'RECIPE_GENERATOR')
  - `title`: Optional conversation title for easy reference
  - `is_canvas_created`: Flag indicating if a canvas visualization was created
  - `canvas_content`: Optional storage for chemical structure visualizations

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

This implementation maintains workspace isolation through the `workspace_id` field and supports the organizational memory requirements by storing complete conversation histories that can be analyzed for knowledge retrieval.
