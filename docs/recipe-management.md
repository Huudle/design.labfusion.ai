# Working with Recipes via Chat

This guide explains how users interact with the AI Agent to create and manage chemical recipes using natural language commands, while adhering to safety protocols and company standards.

## Interaction Model

- Users interact primarily through a **chat interface**.
- The **AI Agent** interprets commands, performs actions by calling backend services, and provides responses.
- A minimal GUI might display complex information (like a full recipe) when requested or presented by the agent.

## Core Concepts (via Agent)

- **Recipe**: Users define recipes by describing them to the agent (e.g., "Create a recipe named X with ingredients Y and Z...").
- **Status**: The agent manages recipe status ("Draft," "Ready to Sell"). Users can ask the agent to change the status (subject to permissions).
- **Versioning**: Edits made via agent commands automatically create new versions, tracked by the Recipe Service.

## Creating a New Recipe (Example Conversation with Checks)

**User**: "Create a new recipe for Basic Aspirin Synthesis."
**Agent**: "Okay, I can start a new recipe called 'Basic Aspirin Synthesis'. What are the ingredients?"
**User**: "Add salicylic acid and acetic anhydride."
**Agent**: "Please specify quantities and units for salicylic acid and acetic anhydride. For example, '100g salicylic acid'. [Standard Enforcement - Quantity/Units]"
**User**: "100g salicylic acid and 150ml acetic anhydride."
**Agent**: "Got it. Added 100g salicylic acid and 150ml acetic anhydride. Warning: Acetic anhydride is corrosive. Ensure proper PPE. [Proactive Safety Check] What are the steps?"
**User**: "Step 1: Mix ingredients. Step 2: Heat for 15 mins."
**Agent**: "Please specify the heating temperature for Step 2. [Standard Enforcement - Missing Info]"
**User**: "Heat to 60C."
**Agent**: "Okay, steps added: 1. Mix ingredients. 2. Heat to 60C for 15 mins. Recipe 'Basic Aspirin Synthesis' created as a draft. Remember the safety warning for acetic anhydride."

- **For Newcomers**: The agent might proactively offer tips or ask clarifying questions related to standards (e.g., "Should we add a standard purification step?").

## Editing and Updating Recipes (Example Conversation with Checks)

**User**: "Add 5 drops of Hydrochloric Acid as catalyst to Basic Aspirin Synthesis draft."
**Agent**: "Please specify the concentration for Hydrochloric Acid (e.g., '12M HCl'). [Standard Enforcement - Concentration]"
**User**: "Use 1M HCl."
**Agent**: "Okay. Analyzing addition... Adding 1M HCl is generally compatible, but ensure you add it slowly. [Proactive Safety Check] I've added 5 drops of 1M Hydrochloric Acid to the ingredients. A new version has been saved."

- **Permissions**: The agent checks user roles. If a Newcomer tries to edit a non-draft recipe they don't own, the agent will respond, "Sorry, as a Newcomer, you can only edit your own draft recipes."

## Searching and Querying Recipes & Knowledge (Example Conversation)

**User**: "Show me recipes using salicylic acid created by experienced chemists in the last year."
**Agent**: "Okay, searching recipes... I found 2 recipes matching your criteria: 'Advanced Aspirin' (Status: Ready to Sell), 'Salicylic Ester Test' (Status: Tested)."
**User**: "What were the main safety concerns discussed for recipes involving exothermic reactions in the past 6 months?"
**Agent**: "Searching conversation logs and experiment notes... Common themes include ensuring adequate cooling capacity, monitoring temperature closely, and planning for potential pressure buildup. Specific concerns were raised about runaway reactions in experiments E-045 and E-051. [Advanced Knowledge Retrieval]"

## Viewing Recipe Details

**User**: "Show me the full details for 'Basic Aspirin Synthesis'."
**Agent**: "Okay, here are the details for 'Basic Aspirin Synthesis': [Agent might display the recipe in a formatted card in the chat, or provide a link to a view in the minimal GUI]."

## Safety Data Sheets (SDS) Integration (Example Conversation)

**User**: "What are the handling precautions for acetic anhydride in the aspirin recipe?"
**Agent**: "Acetic anhydride requires handling in a fume hood with gloves and eye protection. [Agent retrieves info from Material SDS via SDS Service]. Would you like me to generate the full SDS for the 'Basic Aspirin Synthesis' recipe?"
**User**: "Yes, generate the SDS."
**Agent**: "Okay, I've initiated the SDS generation for 'Basic Aspirin Synthesis'. An Experienced Chemist will need to review and approve it. I'll notify you when it's ready for review [If user is Experienced Chemist] / I'll notify an Experienced Chemist for review [If user is Newcomer]."

## Recipe Status and Finalization (Example Conversation)

**User [Experienced Chemist]**: "Review the SDS for 'Basic Aspirin Synthesis'."
**Agent**: "Okay, displaying the generated SDS for 'Basic Aspirin Synthesis' [Shows SDS content/link]. Do you approve this SDS?"
**User [Experienced Chemist]**: "Approve SDS and set recipe to 'Ready to Sell'."
**Agent**: "Before finalizing, please confirm: Have all necessary standard validation checks passed for 'Basic Aspirin Synthesis'? Have potential hazards identified during creation been addressed? [Standard Enforcement / Safety Checkpoint]"
**User [Experienced Chemist]**: "Yes, all checks passed and hazards addressed."
**Agent**: "Confirmed. The SDS for 'Basic Aspirin Synthesis' is approved, status is 'Ready to Sell'. This action has been logged. [Auditing]"
