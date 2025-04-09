

### Core Purpose
The web app will:
1. **Enable chemists to create and refine chemical recipes for sale** (primary goal).
2. **Educate novice chemists** with a focus on safety and recipe design.
3. **Create an organizational memory** to store and leverage collective knowledge.

---

### Functional Requirements

#### 1. Recipe Management
- **Purpose**: Core for creating sellable recipes and preserving organizational memory.
- **Requirements**:
  - Create, edit, and save recipes with: Name, Description, Ingredients (materials + quantities), Steps, Conditions, Yield, Cost, Creator, Date, Status (e.g., “Draft,” “Ready to Sell”).
  - Search/filter recipes by name, ingredient, yield, cost, or creator.
  - Store all recipe versions (organizational memory).
- **For Novices**: Tooltips/guides on recipe design (e.g., “Steps should be clear and repeatable”).

#### 2. Inventory & Pricing Management
- **Purpose**: Support recipe creation and costing; part of organizational memory.
- **Requirements**:
  - Track materials: Name, Category, Quantity, Unit, Location, Price, Supplier, Stock Date, Expiration Date.
  - Update inventory when used in recipes/tests; notify for low/expired stock.
  - Calculate recipe costs from material prices.
- **For Novices**: Explain pricing (e.g., “This affects your recipe’s total cost”).

#### 3. Experiment (Recipe Testing)
- **Purpose**: Validate recipes and build a knowledge base.
- **Requirements**:
  - Log tests: Recipe ID, Date, Researcher, Materials Used, Results, Notes.
  - Link to parent recipe; searchable for trends (organizational memory).
- **For Novices**: Sample entries with design tips (e.g., “Test small batches first”).

#### 4. Safety Data Sheets (SDS)
- **Purpose**: Ensure safety compliance, educate on safety, and archive knowledge.
- **Requirements**:
  - **Material SDS**: Store for each material (Hazards, Handling, Storage, First Aid, Disposal).
  - **Generated SDS**: AI generates key sections for recipes:
    - Hazards, Handling Precautions, Storage Requirements, First Aid Measures, Disposal Instructions.
    - Includes confidence score; editable by chemists.
  - Access SDS from recipes/inventory; archive versions (organizational memory).
- **For Novices**: Emphasize safety with explanations (e.g., “Corrosive means it can burn skin”).

#### 5. AI Querying System
- **Purpose**: Assist recipe creation, educate, and leverage memory.
- **Requirements**:
  - Answer questions on recipes, inventory, costs, SDS, and tests (e.g., “What’s safe handling for Recipe X?”).
  - Simple responses for novices; log queries/answers (organizational memory).
- **For Novices**: Suggest safety/design queries (e.g., “Ask: How do I make this safer?”).

#### 6. User Management
- **Purpose**: Control access and support education.
- **Requirements**:
  - User accounts: Name, Role (“Novice,” “Experienced”), Email/Login.
  - Permissions: Novices view/edit own drafts, access SDS; Experienced edit/finalize SDS, approve recipes.
  - Track creators/editors (organizational memory).

#### 7. Educational Features
- **Purpose**: Focus on teaching novices safety and recipe design.
- **Requirements**:
  - **Safety Education**:
    - Inline guidance on SDS (e.g., “Why gloves matter for corrosives”).
    - Tutorials: “Handling Hazardous Materials,” “Understanding SDS.”
  - **Recipe Design Education**:
    - Tips in recipe editor (e.g., “Precise steps improve repeatability”).
    - Tutorials: “Building a Recipe,” “Optimizing Yield.”
  - “Learn” section: Glossary (e.g., “Hazard: A potential danger”), case studies from past recipes/tests.
  - AI offers educational feedback (e.g., “Add ventilation—Recipe X is flammable”).

#### 8. Organizational Memory
- **Purpose**: Preserve and utilize knowledge.
- **Requirements**:
  - Store recipes, tests, SDS, queries indefinitely; track changes.
  - Search across all data (e.g., “Recipes with acetone, 2024”).
  - Export data (e.g., recipe + SDS) for records.

#### 9. Customer-Facing Recipe Summaries
- **Purpose**: Enhance sellability of recipes.
- **Requirements**:
  - Generate summaries for each “Ready to Sell” recipe:
    - Name, Description (e.g., “High-yield aspirin synthesis”).
    - Key Benefits (e.g., “90% yield, $5 cost per batch”).
    - Simplified SDS Highlights (e.g., “Safe with standard precautions”).
  - Export as PDF or shareable link for customers.
  - Editable by experienced chemists to tailor for sales.
- **For Novices**: Guidance on summaries (e.g., “Highlight what makes your recipe valuable”).

---

### Non-Functional Requirements
- **Accessibility**: Web-based, mobile-friendly.
- **Usability**: Intuitive for novices (guides, clear labels); efficient for experienced users (quick access).
- **Performance**: Fast AI queries (<5 sec), real-time inventory updates.
- **Scalability**: Handle growing recipes/users (100s+).
- **Security**: Role-based access, backups, secure logins.
- **Reliability**: Editable AI SDS with confidence scores for accuracy.

---

### Why This Meets Your Goals
- **Selling Recipes**: Recipe management, testing, costing, SDS, and customer summaries create a complete pipeline for marketable products.
- **Educating Novices**: Safety and recipe design focus in guides, tutorials, and AI feedback builds skills.
- **Organizational Memory**: Versioning, search, and archiving ensure knowledge persists.

---
