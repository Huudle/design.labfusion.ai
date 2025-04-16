# Architecture

## Architectural Style

**OpenAI Agents-Driven Architecture** with **Backend Microservices**:

- **OpenAI Agents Framework**: Primary orchestration layer using OpenAI's Agents Python SDK for agent configuration, handoffs, guardrails, and tracing.
- **Conversational AI**: Primary user interaction via chat, leveraging multi-agent workflows, guardrails, and workspace-specific instructions.
- **Microservices**: Backend logic encapsulated in distinct services, accessed through agent tools.
- **Serverless**: Backend services and AI components deployed using serverless functions.

## System Architecture

```
[Client Layer]
   Chat Interface (Web App - Next.js)
   └── Minimal GUI for Visualization (Optional)

[API Gateway]
   RESTful API (Primarily for /chat endpoint)

[Agent Orchestration Layer]
   ├── Agent Runner (manages agent execution, loops, and handoffs)
   │   ├── Workspace Agent (configured with workspace-specific instructions)
   │   ├── Recipe Agent (specialized for recipe management)
   │   ├── Safety Agent (specialized for safety checks and SDS)
   │   └── Knowledge Agent (specialized for answering complex questions)
   ├── Guardrails (input/output validation, safety checks)
   └── Tracing (logs, spans, debugging)

[Built-in OpenAI Tools]
   ├── WebSearchTool (search for chemical compounds, safety protocols, etc.)
   └── FileSearchTool (retrieve information from vector stores of chemistry documents)

[Custom Function Tools]
   ├── Recipe Tools (CRUD operations for recipes)
   ├── Inventory Tools (material management)
   ├── SDS Generation Tools (safety data sheets)
   ├── User Tools (profile management)
   └── Knowledge Tools (complex queries across data sources)

[Data Layer]
   ├── PostgreSQL Database (Supabase)
   │    ├── Tables: profiles, workspaces, materials, material_sds, recipes, generated_sds, experiments, conversations, standards_rules, audit_log
   │    └── Metadata only: SDS references, document metadata (actual PDFs stored in OpenAI Vector Stores)
   ├── OpenAI Vector Stores
   │    ├── Document Storage: SDS PDFs, recipe PDFs, lab notes, technical documents
   │    └── Vector Embeddings: For semantic search via FileSearchTool
   └── External Data / AI Models (OpenAI APIs)
```

## Components & Responsibilities

### 1. Client Layer (Front-End)

- **Tech**: Next.js
- **Primary Component**: **Chat Interface**
  - Accepts user text input (commands, questions).
  - Sends messages to the API Gateway.
  - Displays responses from the Agent Orchestration Layer.
- **Optional Component**: Minimal GUI for visualizing complex data (e.g., displaying a full recipe table or SDS document presented by the agent).

### 2. API Gateway

- **Tech**: Next.js API Routes or dedicated gateway.
- **Primary Role**: Routes incoming chat messages (e.g., from `/api/chat`) to the Agent Orchestration Layer.
- Handles authentication (via Supabase Auth).

### 3. Agent Orchestration Layer

#### Agent Runner

- **Purpose**: Manages the execution of agents, including the agent loop, responses, tool calls, and handoffs.
- **Technology**: OpenAI Agents Python SDK (`Runner` class)
- **Responsibilities**:
  - Executes the agent loop until final output is produced
  - Handles tool call execution
  - Manages agent-to-agent handoffs
  - Returns final agent output to API Gateway

#### Workspace Agent

- **Purpose**: Primary entry point for all user interactions, configured with workspace-specific instructions.
- **Technology**: OpenAI Agents Python SDK (`Agent` class)
- **Configuration**:
  - **Instructions**: Loaded from workspace's `custom_instructions` field
  - **Tools**: Access to all required tools for basic interactions
  - **Handoffs**: Access to specialized agents for specific tasks
- **Responsibilities**:
  - Initial triage of user requests
  - Handling basic queries directly
  - Handing off to specialized agents for complex tasks

#### Specialized Agents

- **Recipe Agent**:
  - **Instructions**: Specialized in recipe creation, modification, and validation
  - **Tools**: Recipe-specific tools, inventory lookup tools, WebSearchTool for chemical information
  - **Guardrails**: Enforces recipe standards and safety protocols
- **Safety Agent**:
  - **Instructions**: Focused on safety checks, warnings, and SDS management
  - **Tools**: SDS generation tools, material compatibility checks, WebSearchTool for safety information
  - **Guardrails**: Strict validation for safety-critical operations
- **Knowledge Agent**:
  - **Instructions**: Optimized for answering complex questions across data sources
  - **Tools**: Knowledge retrieval tools, experiment lookup, FileSearchTool for document retrieval
  - **Output Type**: Potentially structured for complex answers with citations

#### Guardrails

- **Purpose**: Provides configurable safety checks for input and output validation.
- **Implementation**: Using OpenAI Agents SDK guardrails system
- **Types**:
  - **Input Validation**: Checks user inputs for proper format and safety
  - **Output Validation**: Ensures agent responses meet quality and safety standards
  - **Domain-Specific Checks**: Custom checks for chemical safety, standards compliance

#### Tracing

- **Purpose**: Tracks agent runs, allowing for debugging, optimization, and auditability.
- **Implementation**: Using OpenAI Agents SDK built-in tracing
- **Features**:
  - Records complete agent conversations, tool calls, and handoffs
  - Provides debugging interfaces for troubleshooting
  - Exports traces to external systems (optional)
  - Serves as input to audit log

### 4. Built-in OpenAI Tools

#### WebSearchTool

- **Purpose**: Enables agents to search the web for relevant information.
- **Use Cases**:
  - Searching for up-to-date information about chemical compounds
  - Finding recent safety protocols and best practices
  - Locating regulatory information about materials
  - Supplementing internal knowledge with external references
- **Implementation**: Integrated directly using OpenAI's built-in tool

#### FileSearchTool

- **Purpose**: Retrieves information from vector stores containing chemistry-related documents.
- **Use Cases**:
  - Searching through company SOPs and guidelines
  - Finding relevant information in scientific papers and reference materials
  - Retrieving historical recipes and experiment results
  - Accessing chemical reference data
- **Implementation**:
  - Connected to OpenAI Vector Stores containing uploaded chemistry documents
  - Configured with appropriate filters and settings for chemical domain

### 5. Custom Function Tools

Each service is implemented as a collection of custom function tools that can be called during the agent loop:

#### Recipe Tools

- **Implemented as**: Function tools using the `@function_tool` decorator
- **Operations**: Create, read, update, delete recipes
- **Validation**: Includes tool-level validation logic for recipe structure

#### Inventory Tools

- **Operations**: Track materials, check stock levels, update quantities
- **Integration**: May call external inventory systems

#### SDS Generation Tools

- **Operations**: Generate safety data sheets, retrieve material SDS
- **Integration**: May leverage LLM capabilities for synthesis

#### User Tools

- **Operations**: User profile management, preferences
- **Security**: Strict permission checks for user-related operations

#### Knowledge Tools

- **Operations**: Complex queries across multiple data sources
- **Implementation**: May use specialized retrieval techniques

### 6. Data Layer

- **Tech**: Supabase (PostgreSQL + Storage) and OpenAI Vector Stores
- **Database**: Stores core entities with workspace isolation via RLS policies
  - `workspaces`: Contains workspace metadata and custom instructions
  - `profiles`: User data linked to auth.users
  - Other domain tables with workspace_id for isolation
  - **Document Metadata**: References to documents stored in vector stores
- **OpenAI Vector Stores**: Primary storage for all document content
  - **Document Types**:
    - Safety Data Sheets (SDS)
    - Technical documentation and manuals
    - Scientific papers and reference materials
    - Lab notes and research reports
    - Company guidelines and SOPs
  - **Benefits**:
    - Semantic search capabilities via FileSearchTool
    - Automatic content indexing for knowledge retrieval
    - Integration with AI agents for document understanding
- **External Services**: OpenAI API for model access

## Multi-Agent Workflow Examples

### 1. Recipe Creation with WebSearch Integration

1. **Client**: User sends message "Create a new recipe for polymer synthesis using acrylonitrile"
2. **API Gateway**: Routes message to Agent Runner
3. **Agent Runner**:
   - Starts with Workspace Agent (configured with workspace-specific instructions)
   - Workspace Agent recognizes recipe creation intent
   - Workspace Agent hands off to Recipe Agent
4. **Recipe Agent**:
   - Collects necessary details from user
   - Makes tool calls to check inventory for acrylonitrile
   - Uses WebSearchTool to find current best practices for acrylonitrile polymerization
   - Recognizes safety concerns with acrylonitrile
   - Hands off to Safety Agent for safety assessment
5. **Safety Agent**:
   - Retrieves SDS information for acrylonitrile
   - Uses WebSearchTool to check for recent safety advisories about acrylonitrile
   - Performs compatibility checks
   - Provides safety warnings and recommendations
   - Hands back to Recipe Agent
6. **Recipe Agent**:
   - Creates the recipe with safety information incorporated
   - Returns final formatted recipe to user
7. **Tracing**: Records the entire multi-agent workflow, tool calls, and handoffs

### 2. Knowledge Retrieval with FileSearchTool

1. **Client**: User asks "What were the yields of our acetone reactions last month and why did batch 3 fail?"
2. **Workspace Agent**:
   - Recognizes complex knowledge query
   - Hands off to Knowledge Agent
3. **Knowledge Agent**:
   - Makes tool calls to query experiments database
   - Uses FileSearchTool to search through relevant lab notes and reports about acetone reactions
   - Analyzes results across multiple experiments
   - Synthesizes explanation for batch 3 failure
   - Provides structured response with data and analysis
4. **Tracing**: Captures the reasoning process and data sources used

## Integration with Workspace Custom Instructions

The system leverages workspace-specific custom instructions stored in the `workspaces` table:

1. **Initialization**: When a user session starts, the system:

   - Retrieves the user's assigned workspace
   - Loads the `custom_instructions` field from that workspace
   - Creates a Workspace Agent configured with these instructions

2. **Instruction Types**: Workspace instructions can include:

   - Tone and style guidelines
   - Company-specific terminology
   - Safety protocols and priorities
   - Domain-specific rules and standards
   - Preferred formats for recipes and reports
   - Guidelines for when to use WebSearchTool and FileSearchTool

3. **Agent Behavior**: All agents in the workspace context will:
   - Follow the workspace-specific instructions
   - Maintain consistent terminology
   - Apply appropriate safety standards
   - Use formats specific to the organization

## Workspace-Specific Agent Configuration

To provide maximum flexibility, the system supports configuring different agent behaviors and capabilities for each workspace:

### Agent Configuration Storage

1. **Enhanced Workspace Schema**:

   - The `workspaces` table includes configuration fields for agents:

   ```sql
   ALTER TABLE workspaces ADD COLUMN agent_configurations JSONB DEFAULT '{}'::jsonb;
   ```

   - This JSONB field stores workspace-specific agent configurations:

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

2. **Configuration Interface**:
   - Admin users can configure agent settings through a dedicated interface
   - Settings can include: model selection, temperature, tool availability, specializations
   - Changes are versioned for audit purposes

### Dynamic Agent Creation

1. **Agent Factory Pattern**:

   - A factory module dynamically creates agents based on workspace configuration:

   ```python
   def create_workspace_agents(workspace_id):
       # Fetch workspace configuration
       workspace = fetch_workspace(workspace_id)
       agent_configs = workspace.agent_configurations

       # Create the primary workspace agent
       workspace_agent = Agent(
           name="Workspace Agent",
           instructions=workspace.custom_instructions,
           model=agent_configs.get("workspace_agent", {}).get("model", "gpt-4"),
           temperature=agent_configs.get("workspace_agent", {}).get("temperature", 0.2),
           tools=get_enabled_tools(agent_configs.get("workspace_agent", {}).get("enabled_tools", []))
       )

       # Create specialized agents based on configuration
       recipe_agent = create_recipe_agent(workspace, agent_configs)
       safety_agent = create_safety_agent(workspace, agent_configs)
       knowledge_agent = create_knowledge_agent(workspace, agent_configs)

       # Set up handoffs
       workspace_agent.handoffs = [recipe_agent, safety_agent, knowledge_agent]

       return {
           "workspace_agent": workspace_agent,
           "recipe_agent": recipe_agent,
           "safety_agent": safety_agent,
           "knowledge_agent": knowledge_agent
       }
   ```

2. **Session Initialization**:
   - When a user starts a session, agents are created based on their workspace configuration
   - Agents are cached for performance but refreshed when configurations change

### Customization Options

1. **Model Selection**:

   - Each workspace can specify different OpenAI models for different agents
   - High-stakes agents (like Safety) might use more capable models
   - Cost-conscious workspaces might use more efficient models for routine tasks

2. **Tool Availability**:

   - Workspaces can enable/disable specific tools per agent
   - Example: A workspace might enable WebSearchTool only for the Knowledge Agent
   - Access to sensitive tools can be restricted for specific workspaces

3. **Specialized Instructions**:

   - Beyond the general `custom_instructions`, each agent can have specific instructions
   - These can tailor the agent's behavior for the workspace's domain
   - Example: Safety Agent in a pharmaceutical workspace vs. a polymer manufacturing workspace

4. **Custom Agent Types**:
   - Workspaces can have domain-specific agent types
   - Example: A pharmaceutical workspace might have a "Regulatory Agent" specialized in compliance
   - These are created based on configuration flags in the workspace settings

### Configuration Examples

1. **Research Laboratory Workspace**:

   - Knowledge Agent configured with academic paper access
   - Recipe Agent optimized for experimental protocols
   - Safety Agent with research lab safety guidelines

2. **Manufacturing Workspace**:

   - Recipe Agent focused on scalability and efficiency
   - Safety Agent enforcing industrial safety standards
   - Custom "Production Agent" for handling manufacturing workflows

3. **Pharmaceutical Workspace**:
   - Custom "Regulatory Agent" for compliance workflows
   - Recipe Agent specialized in pharmaceutical synthesis
   - Safety Agent with enhanced medical safety protocols

### Implementation Considerations

1. **Default Configurations**:

   - System provides sensible defaults for all agent settings
   - Newly created workspaces inherit these defaults
   - Templates available for common workspace types (research, manufacturing, etc.)

2. **Agent Versioning**:

   - Changes to agent configurations are versioned
   - Critical workflows can be pinned to specific agent versions
   - Rollback capability for configuration changes

3. **Configuration Validation**:
   - System validates that agent configurations are valid
   - Prevents invalid combinations of settings
   - Enforces security policies (e.g., some tools might require admin approval)

## Vector Store Management

To support the FileSearchTool, the system includes a process for maintaining and updating vector stores:

1. **Document Processing**:

   - Chemistry-related documents (SDS, technical docs, etc.) are uploaded to the system
   - Documents are processed and stored directly in OpenAI Vector Stores
   - Metadata records are created in Supabase for tracking and organization
   - Documents are tagged with appropriate metadata (document type, chemistry domain, date)

2. **Search Configuration**:

   - FileSearchTool is configured with appropriate filters and relevance settings
   - Search results are filtered based on workspace context
   - Results include source citations and confidence scores

3. **Document Management Workflow**:
   - New documents uploaded through application interface
   - PDF contents stored directly in OpenAI Vector Stores (not in Supabase)
   - Only metadata and references stored in Supabase database
   - Vector embeddings automatically generated during upload
   - Documents available immediately for semantic search via FileSearchTool

## Non-Functional Considerations

- **Scalability**: Agent operations are stateless and can scale horizontally.
- **Performance**:
  - Agent handoffs may introduce latency
  - Tool execution time is critical to overall responsiveness
  - Vector store queries optimized for speed and relevance
- **Reliability**:
  - Tracing enables comprehensive debugging
  - Agent retry mechanisms for transient failures
  - Fallback strategies when web search or vector search fails
- **Security**:
  - Strong isolation between workspaces
  - Permission checks at tool execution level
  - Secure handling of external search results
- **Maintainability**:
  - Clear separation between agent configuration and tool implementation
  - Extensible architecture for adding new specialized agents
  - Modular tool integration allowing for easy updates
- **Safety**:
  - Multiple layers of guardrails
  - Explicit handoffs to Safety Agent for critical operations
  - Validation of information from web searches against internal safety standards
- **Auditability**:
  - Comprehensive tracing of all agent actions
  - Integration with audit log for compliance
  - Records of all web searches and document retrievals
