# Authentication & Workspace Flow (MVP)

## Overview

This document outlines the authentication flow for the labXpert.ai platform's first version (MVP). The system implements a simplified multi-tenant architecture where workspaces are pre-created by developers. Authentication is powered by [Supabase Auth](https://supabase.com/docs/guides/auth).

## Authentication Flow with Supabase

### Sign-Up Process

1. **User Registration**

   - New users provide their name, email, and password through the Supabase Auth UI or API
   - Supabase validates input and checks for existing accounts
   - Account is created with appropriate security measures

2. **Email Verification**

   - Supabase automatically sends verification email with secure link
   - User clicks link to confirm email ownership
   - Account status is updated in Supabase's auth tables

3. **Workspace Assignment**

   - In this first version, users are pre-assigned to workspaces by developers
   - Each user is associated with exactly one workspace
   - The workspace assignment is stored in user metadata

### Sign-In Process

1. **Supabase Authentication**

   - User signs in through Supabase Auth UI or API
   - Credentials validated against Supabase Auth tables
   - On successful authentication:
     - Supabase issues JWT token with user information
     - User is automatically directed to their pre-assigned workspace

2. **Session Management**
   - Supabase handles token lifecycle (expiration, refresh)
   - Session persistence options managed through Supabase configuration

## Workspace Implementation (Developer-Managed)

### Pre-Creating Workspaces

1. **Development Process**

   - Developers create workspaces directly in the database or through admin interfaces
   - Each workspace has basic configuration for its intended team/organization
   - No user-facing interface for workspace creation in MVP

2. **User-Workspace Association**
   - Developers assign users to workspaces upon account creation
   - User's workspace context is stored in Supabase user metadata
   - All user data and interactions are scoped to their assigned workspace

## Security Considerations

### Supabase Auth Security

- Password policies managed by Supabase
- Multi-factor authentication available through Supabase Auth
- Account lockout and protection against brute force attacks
- Secure credential storage handled by Supabase (Postgres + pgcrypto)

### Workspace Isolation

- Data separation enforced at database level with Row Level Security (RLS)
- RLS policies in Supabase ensure users can only access their workspace data
- Cross-workspace data access explicitly prohibited through RLS

### Audit Trail

- Authentication events logged in Supabase auth schema
- Additional workspace-specific events logged in application audit tables

## Technical Implementation

### Supabase Authentication Integration

```javascript
// Initialize Supabase client
const supabaseClient = createClient(SUPABASE_URL, SUPABASE_KEY);

// Sign up a new user (developer process)
const { user, error } = await supabaseClient.auth.signUp({
  email: "user@example.com",
  password: "secure-password",
  options: {
    data: {
      name: "John Doe",
      assigned_workspace_id: "pre-created-workspace-id",
    },
  },
});

// Sign in a user
const { data, error } = await supabaseClient.auth.signInWithPassword({
  email: "user@example.com",
  password: "secure-password",
});

// Sign out
await supabaseClient.auth.signOut();
```

### Developer Workspace Management

```javascript
// Admin API: Create a new workspace (developer only)
const { data, error } = await adminSupabaseClient
  .from("workspaces")
  .insert([
    { name: "Research Lab", description: "Chemistry research workspace" },
  ])
  .select();

// Admin API: Assign user to workspace (developer only)
const { data, error } = await adminSupabaseClient.auth.admin.updateUserById(
  userId,
  {
    user_metadata: {
      assigned_workspace_id: workspaceId,
    },
  }
);
```

### Row Level Security (RLS) Policies

```sql
-- Materials table RLS policy
CREATE POLICY "Users can only access materials in their assigned workspace"
ON materials
FOR ALL
USING (
  workspace_id = (auth.jwt() -> 'user_metadata' ->> 'assigned_workspace_id')::uuid
);
```

## Integration with Conversational AI

The AI agent operates within the context of the user's assigned workspace:

1. **Session Context**

   - AI agent receives user's JWT from Supabase with each request
   - JWT contains assigned workspace context
   - All data retrieval and mutations respect workspace boundaries through RLS

2. **Simplified Role System**

   - In the MVP, all users have the same permissions within their workspace
   - Future versions may introduce role-based access control

## Future Enhancements (Post-MVP)

The following features are planned for future releases:

1. **Self-Service Workspace Creation**

   - Allow users to create their own workspaces
   - Add workspace customization options

2. **Invitation System**

   - Implement workspace invitations
   - Email-based user onboarding

3. **Role Management**

   - Introduce admin/member roles
   - Add role-specific permissions
   - Support for "Newcomer" and "Experienced" chemistry roles

4. **Multi-Workspace Support**
   - Allow users to belong to multiple workspaces
   - Add workspace switching interface
