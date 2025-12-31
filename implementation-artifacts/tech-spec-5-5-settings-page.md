# Tech Spec: Story 5.5 - Settings Page

**Story**: [5-5-settings-page.md](5-5-settings-page.md)
**Status**: Draft

## 1. Objective
Implement a comprehensive Settings Page that allows users to configure the Runtime environment. Key areas are **LLM Provider Configuration** (essential for execution) and **Package Cache Management**.

## 2. Technical Architecture

### 2.1 Backend (`RuntimeStore`)
- **State**:
    - `settings.llm`: `{ provider, model, baseUrl, apiKey }`
    - `settings.theme`: `'system' | 'light' | 'dark'`
    - `cachedPackages`: Array of package metadata.
- **Persistence**: Saved to `userData/runtime-store/settings.json`.
- **Modifications**:
    - Existing `updateSettings` method is generic. No backend changes required for config.
    - `packages` management uses existing `runtimeStore` methods.

### 2.2 IPC Layer (`preload.ts`)
- **Existing**: `settings:get`, `settings:update`.
- **Existing**: `packages:list`, `packages:remove`, `packages:clear`, `package:import`.
- **New**: None required. Existing IPC covers requirements.

### 2.3 Frontend (React)
- **Component**: `SettingsPage.tsx` refactor.
- **Structure**:
    - **Navigation**: Side tabs or Top tabs for sections (General, LLM, Packages).
    - **Section 1: General**: Theme toggle.
    - **Section 2: LLM**:
        - Form: Provider (Select), Base URL (Input, conditional), Model (Input), API Key (Password Input).
        - Actions: "Save" (persists to store), "Test Connection" (ephemeral check).
    - **Section 3: Packages**:
        - List view of `cachedPackages` (from `useAppStore`).
        - Import `.bmad` button.
        - Delete/Clear actions.

## 3. Data Models

### 3.1 LLM Configuration
```typescript
interface LLMConfig {
  provider: 'openai' | 'openai-compatible' | 'ollama' | 'azure';
  baseUrl: string; // Required for compatible/ollama
  model: string;    // e.g. "gpt-4", "deepseek-chat"
  apiKey: string;  // Optional for Ollama
  timeout: number;
}
```

### 3.2 Key logic
- **Test Connection**: Since the backend `LLMServiice` isn't fully exposed via IPC for "testing" yet (part of Epic 4), the frontend can simulate a test or we can add a lightweight `llm:test` IPC handler later. For MVP, we can perform a simple fetch from Renderer if CORS allows, otherwise mock it. *Decision: Mock success for UI flow, mark as TODO for real implementation.*
- **Package Binding**: When a package is removed, check if `activeProject.activePackageId` matches. If so, trigger the "Package Missing" warning state (handled by `useAppStore`).

## 4. Testing Strategy

### 4.1 Manual Verification
1. **LLM Config**:
    - Select "OpenAI Compatible".
    - Enter DeepSeek URL & Key.
    - Click "Test" -> Show "Success".
    - Click Save -> Restart App -> Verify persistence.
2. **Packages**:
    - Import a valid `.bmad` file -> Appears in list.
    - Click Delete -> Dissapears from list.
3. **Theme**:
    - Toggle Light/Dark -> UI updates immediately.

### 4.2 Automated Tests
- Unit tests for `RuntimeStore.updateSettings` (already exists).

## 5. Implementation Plan
1. **Refactor `SettingsPage`**: Split current monolithic page into Sections/Tabs.
2. **Implement LLM Form**: Add state management for form fields + validation.
3. **Implement Package List**: Move current "Package Management" UI into the Packages tab.
4. **Wire Actions**: Connect "Test Connection" (mock) and "Save".
