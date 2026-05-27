# Keycloak Workspace Access Model

This repository now boots Keycloak with a pre-imported realm designed for a workspace-based multi-tenant app.

## What is configured

- Realm: `workspace-realm`
- Bootstrap super admin user: `superadmin`
- Workspace role model:
  - `super_admin`
  - `workspace_admin`
  - `workspace_user`
  - `workspace_viewer`
  - `workspace_editor`
  - `cross_workspace_delegate`
  - `impersonation_operator`
- Default workspace tree:
  - `/workspaces/ws-default/admins`
  - `/workspaces/ws-default/members`
  - `/workspaces/ws-default/viewers`
  - `/workspaces/ws-default/editors`
- Invitation root group: `/invites`
- Clients:
  - `workspace-web` (public browser app)
  - `workspace-api` (confidential backend API)

## Start Keycloak

```bash
docker compose up -d
```

Keycloak runs at `http://localhost:8081`.

## Requirement mapping

### 1) App authentication via Keycloak

Use OIDC with client `workspace-web` for login and `workspace-api` for backend token validation.

### 2) App should have workspaces

Each workspace is represented as a group under `/workspaces`, for example:
- `/workspaces/ws-acme`

Each workspace can include standard subgroups:
- `/workspaces/ws-acme/admins`
- `/workspaces/ws-acme/members`
- `/workspaces/ws-acme/viewers`
- `/workspaces/ws-acme/editors`

### 3) First user is super admin

Imported bootstrap account:
- username: `superadmin`
- role: `super_admin`
- group: `/workspaces/ws-default/admins`
- additional admin privileges for user management and impersonation

### 4) Super admin can send invitations

Recommended backend flow:
1. Create or update user with invited email.
2. Assign pending metadata in attributes or create an invite subgroup under `/invites`.
3. Add user to target workspace subgroup (`admins`, `members`, `viewers`, `editors`).
4. Trigger Keycloak execute-actions email (`UPDATE_PASSWORD`, optional `VERIFY_EMAIL`).

### 5) Each workspace has its own admin

Workspace admins are users in `/workspaces/<workspace_id>/admins` and should hold:
- `workspace_admin`
- `workspace_editor`

### 6) Only super admin can access multiple workspaces by default

Keep normal users only in one workspace branch.
Grant super admins global scope using:
- realm role `super_admin`
- user attribute `workspace_scope=*`

### 7) Super admin can grant others cross-workspace rights

When needed, add user into additional workspace subgroup(s) and assign `cross_workspace_delegate`.
App authorization should validate both:
- membership in target workspace group
- permission role (`workspace_viewer` or `workspace_editor` or `workspace_admin`)

### 8) Impersonation support

Bootstrap super admin already has realm-management roles:
- `impersonation`
- `manage-users`
- `view-users`
- `query-users`
- `query-groups`

This allows backend/admin-console impersonation flows.

## Operational examples (admin API)

Use Keycloak admin REST from `workspace-api` or admin user token.

- Create workspace group:
  - `POST /admin/realms/workspace-realm/groups`
- Create subgroup (`admins`, `members`, etc.):
  - `POST /admin/realms/workspace-realm/groups/{group-id}/children`
- Add user to subgroup:
  - `PUT /admin/realms/workspace-realm/users/{user-id}/groups/{group-id}`
- Send invite email actions:
  - `PUT /admin/realms/workspace-realm/users/{user-id}/execute-actions-email`
- Impersonate user:
  - `POST /admin/realms/workspace-realm/users/{user-id}/impersonation`

## Important follow-up changes

- Change all default secrets in `.env` and `realm-import/workspace-realm.json`.
- Point `workspace-web` redirect URLs to your real frontend domains.
- Implement backend authorization guard using workspace group + role claims.
