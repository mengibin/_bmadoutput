# Story 5.5: Settings Page (LLM & Packages)

Status: ready-for-design

## Story

As a **User**,
I want to **configure global runtime settings**,
so that I can connect to my preferred LLM provider, manage imported packages, and customize the UI appearance.

## Acceptance Criteria

### 1. LLM Configuration
1. **Given** the Settings page
   **When** I select the **LLM Provider** tab
   **Then** I can configure:
     - **Provider**: Dropdown (OpenAI, OpenAI-Compatible, Ollama, Azure).
     - **Base URL**: Text input (visible/editable for Compatible/Ollama).
     - **Model**: Text input (e.g. `gpt-4`, `deepseek-chat`).
     - **API Key**: Password input.
2. **Given** filled configuration
   **When** I click **Test Connection**
   **Then** the app attempts a dry-run call and shows Success/Failure notification.
3. **Given** changes
   **When** I click **Save**
   **Then** the configuration is persisted to `RuntimeStore` (user data).

### 2. Package Management
1. **Given** the Settings page
   **When** I select the **Packages** tab
   **Then** I see a list of cached packages (Name, Version, ID).
2. **Given** the package list
   **When** I click **Import Package**
   **Then** I can select a `.bmad` file, which is validated and added to the cache.
3. **Given** a cached package
   **When** I click **Delete**
   **Then** it is removed from the local cache.
   - *Constraint*: If the package is currently active in an open project, warn the user.
4. **Given** an active project with a missing package
   **When** I verify the package here or re-import it
   **Then** the global "Missing Package" blocking state resolves.

### 3. General / Theme
1. **Given** the Settings page
   **When** I toggle **Theme**
   **Then** the UI switches between Light/Dark/System immediately.

## Technical Context

- **IPC**:
    - `settings:get` / `settings:update` (Generic store access).
    - `package:import` / `packages:list` / `packages:remove` (Existing handlers).
- **State**: `RuntimeStore` persists `llmConfig` and `theme`.
- **Security**: API Keys stored in `RuntimeStore` (local JSON). *Future: Use Electron SafeStorage.*

## Dependencies
- Story 5-0 (Shell/Navigation) - *In Progress* (Settings entry point exists).

## Design
- **Layout**: Sidebar/Tabs for sections (LLM, Packages, General).
- **Feedback**: Toasts for "Saved" and "Test Connection" results.
- **Reference**: See `runtime-development-roadmap.md` (Phase 1, Item 3).
- **Tech Spec**: [_bmad-output/implementation-artifacts/tech-spec-5-5-settings-page.md](tech-spec-5-5-settings-page.md)
