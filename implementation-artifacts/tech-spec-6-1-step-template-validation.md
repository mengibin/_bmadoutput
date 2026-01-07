# Tech Spec: Story 6.1 Step Template Validation & UI Display

## Overview

This spec details the implementation of step file template validation during package import and section-based UI display in the Builder.

## 1. Step File Structure Specification

### 1.1 Required Format

```markdown
---
schemaVersion: "1.1"
nodeId: "step-01"
type: "step"
title: "Step Title"
agentId: "agent_id"
inputs: []
outputs: []
---

# Step Title

## Goal

Description of what this step accomplishes.

## Instructions

Detailed instructions for the LLM agent.

## Completion

(Optional) State update instructions.
```

### 1.2 Validation Rules

| Component | Required | Validation |
|:---|:---|:---|
| YAML Frontmatter | ✅ | Must exist, valid YAML |
| `schemaVersion` | ✅ | Must match `^1\.1(\.\d+)?$` |
| `nodeId` | ✅ | Must match `^[A-Za-z0-9][A-Za-z0-9._:-]*$` |
| `type` | ✅ | One of: step, decision, merge, end, subworkflow |
| `title` | ❌ | String, optional |
| `agentId` | ❌ | String, optional |
| `## Goal` | ✅ | Section must exist with content |
| `## Instructions` | ✅ | Section must exist with content |
| `## Completion` | ❌ | Optional section |

---

## 2. Backend Implementation

### 2.1 New Types

```python
# In app/services/package_import_service.py

@dataclass
class StepSection:
    name: str
    content: str
    line_start: int
    line_end: int

@dataclass
class StepParseResult:
    frontmatter: dict
    sections: dict[str, StepSection]  # Goal, Instructions, Completion
    errors: list[ValidationError]
    warnings: list[str]
```

### 2.2 validate_step_file Function

```python
def validate_step_file(content: str, file_path: str) -> list[ValidationError]:
    """
    Validate step file structure and return errors.
    
    Args:
        content: Raw markdown content
        file_path: Path for error reporting
    
    Returns:
        List of validation errors (empty if valid)
    """
    errors = []
    
    # 1. Parse frontmatter
    frontmatter, body, fm_error = parse_frontmatter(content)
    if fm_error:
        errors.append(ValidationError("STEP_MISSING_FRONTMATTER", file_path, "", fm_error))
        return errors
    
    # 2. Validate required frontmatter fields
    required_fields = ["schemaVersion", "nodeId", "type"]
    for field in required_fields:
        if field not in frontmatter or not frontmatter[field]:
            errors.append(ValidationError(
                "STEP_INVALID_FRONTMATTER", 
                file_path, 
                field, 
                f"Missing required field: {field}"
            ))
    
    # 3. Validate field formats
    if "schemaVersion" in frontmatter:
        if not re.match(r"^1\.1(\.\d+)?$", str(frontmatter["schemaVersion"])):
            errors.append(ValidationError(
                "STEP_INVALID_FRONTMATTER",
                file_path,
                "schemaVersion",
                f"Invalid schemaVersion: {frontmatter['schemaVersion']}"
            ))
    
    if "type" in frontmatter:
        valid_types = ["step", "decision", "merge", "end", "subworkflow"]
        if frontmatter["type"] not in valid_types:
            errors.append(ValidationError(
                "STEP_INVALID_FRONTMATTER",
                file_path,
                "type",
                f"Invalid type '{frontmatter['type']}', must be one of: {valid_types}"
            ))
    
    # 4. Parse and validate sections
    sections = parse_sections(body)
    
    if "Goal" not in sections:
        errors.append(ValidationError(
            "STEP_MISSING_GOAL",
            file_path,
            "",
            "Missing required section: ## Goal"
        ))
    
    if "Instructions" not in sections:
        errors.append(ValidationError(
            "STEP_MISSING_INSTRUCTIONS",
            file_path,
            "",
            "Missing required section: ## Instructions"
        ))
    
    return errors
```

### 2.3 parse_frontmatter Function

```python
def parse_frontmatter(content: str) -> tuple[dict, str, str | None]:
    """
    Parse YAML frontmatter from markdown content.
    
    Returns:
        (frontmatter_dict, body_content, error_message)
    """
    if not content.startswith("---"):
        return {}, content, "No YAML frontmatter found"
    
    # Find closing ---
    end_match = re.search(r"\n---\s*\n", content[3:])
    if not end_match:
        return {}, content, "Unclosed YAML frontmatter"
    
    fm_end = end_match.end() + 3
    fm_content = content[3:fm_end - 4]
    body = content[fm_end:]
    
    try:
        frontmatter = yaml.safe_load(fm_content) or {}
        return frontmatter, body, None
    except yaml.YAMLError as e:
        return {}, content, f"Invalid YAML: {e}"
```

### 2.4 parse_sections Function

```python
def parse_sections(body: str) -> dict[str, StepSection]:
    """
    Parse markdown sections from body content.
    
    Returns:
        Dict mapping section name to StepSection
    """
    sections = {}
    lines = body.split("\n")
    
    current_section = None
    section_start = 0
    section_content = []
    
    for i, line in enumerate(lines):
        # Match ## SectionName or ## SectionName (extra)
        match = re.match(r"^##\s+(\w+)(?:\s+\(.*\))?", line)
        if match:
            # Save previous section
            if current_section:
                sections[current_section] = StepSection(
                    name=current_section,
                    content="\n".join(section_content).strip(),
                    line_start=section_start,
                    line_end=i - 1
                )
            
            current_section = match.group(1)
            section_start = i
            section_content = []
        elif current_section:
            section_content.append(line)
    
    # Save last section
    if current_section:
        sections[current_section] = StepSection(
            name=current_section,
            content="\n".join(section_content).strip(),
            line_start=section_start,
            line_end=len(lines) - 1
        )
    
    return sections
```

### 2.5 Integration in import_package

```python
# In import_package(), after reading step files:

for path, content in wf_steps.items():
    step_errors = validate_step_file(content, path)
    errors.extend(step_errors)
```

---

## 3. Frontend Implementation

### 3.1 Step Parser (src/lib/step-parser.ts)

```typescript
export interface StepFrontmatter {
  schemaVersion: string;
  nodeId: string;
  type: "step" | "decision" | "merge" | "end" | "subworkflow";
  title?: string;
  agentId?: string;
  inputs?: string[];
  outputs?: string[];
  transitions?: Array<{
    to: string;
    label: string;
    isDefault?: boolean;
    conditionText?: string;
  }>;
}

export interface StepSections {
  goal: string;
  instructions: string;
  completion?: string;
}

export interface StepData {
  frontmatter: StepFrontmatter;
  sections: StepSections;
  rawContent: string;
}

export function parseStepMarkdown(content: string): StepData | null {
  // Parse frontmatter
  const fmMatch = content.match(/^---\s*\n([\s\S]*?)\n---\s*\n([\s\S]*)$/);
  if (!fmMatch) return null;
  
  const frontmatter = yaml.load(fmMatch[1]) as StepFrontmatter;
  const body = fmMatch[2];
  
  // Parse sections
  const sections: StepSections = {
    goal: "",
    instructions: "",
  };
  
  const sectionRegex = /^##\s+(Goal|Instructions|Completion)(?:\s+\(.*\))?/gm;
  let matches: RegExpExecArray | null;
  const sectionPositions: Array<{ name: string; start: number }> = [];
  
  while ((matches = sectionRegex.exec(body)) !== null) {
    sectionPositions.push({ name: matches[1], start: matches.index });
  }
  
  sectionPositions.forEach((pos, i) => {
    const start = pos.start + body.slice(pos.start).indexOf("\n") + 1;
    const end = i < sectionPositions.length - 1 
      ? sectionPositions[i + 1].start 
      : body.length;
    const content = body.slice(start, end).trim();
    
    if (pos.name === "Goal") sections.goal = content;
    else if (pos.name === "Instructions") sections.instructions = content;
    else if (pos.name === "Completion") sections.completion = content;
  });
  
  return { frontmatter, sections, rawContent: content };
}

export function serializeStepMarkdown(data: StepData): string {
  const fm = yaml.dump(data.frontmatter, { lineWidth: -1 });
  let body = `# ${data.frontmatter.title || "Step"}\n\n`;
  body += `## Goal\n\n${data.sections.goal}\n\n`;
  body += `## Instructions\n\n${data.sections.instructions}\n`;
  if (data.sections.completion) {
    body += `\n## Completion\n\n${data.sections.completion}\n`;
  }
  return `---\n${fm}---\n\n${body}`;
}
```

### 3.2 StepEditorPanel Component

```tsx
// src/components/StepEditorPanel.tsx

interface StepEditorPanelProps {
  content: string;
  agents: AgentListItem[];
  onChange: (content: string) => void;
  onSave: () => void;
}

export function StepEditorPanel({ content, agents, onChange, onSave }: StepEditorPanelProps) {
  const [stepData, setStepData] = useState<StepData | null>(null);
  const [parseError, setParseError] = useState<string | null>(null);
  
  useEffect(() => {
    const parsed = parseStepMarkdown(content);
    if (parsed) {
      setStepData(parsed);
      setParseError(null);
    } else {
      setParseError("无法解析 Step 文件格式");
    }
  }, [content]);
  
  const handleFrontmatterChange = (field: string, value: any) => {
    if (!stepData) return;
    const updated = {
      ...stepData,
      frontmatter: { ...stepData.frontmatter, [field]: value }
    };
    setStepData(updated);
    onChange(serializeStepMarkdown(updated));
  };
  
  const handleSectionChange = (section: keyof StepSections, value: string) => {
    if (!stepData) return;
    const updated = {
      ...stepData,
      sections: { ...stepData.sections, [section]: value }
    };
    setStepData(updated);
    onChange(serializeStepMarkdown(updated));
  };
  
  if (parseError) {
    // Fallback to raw text editor
    return <textarea value={content} onChange={e => onChange(e.target.value)} />;
  }
  
  return (
    <div className="step-editor">
      {/* Frontmatter Form */}
      <FrontmatterForm 
        data={stepData.frontmatter} 
        agents={agents}
        onChange={handleFrontmatterChange}
      />
      
      {/* Goal Section */}
      <SectionEditor
        title="Goal"
        content={stepData.sections.goal}
        onChange={v => handleSectionChange("goal", v)}
        required
      />
      
      {/* Instructions Section */}
      <SectionEditor
        title="Instructions"
        content={stepData.sections.instructions}
        onChange={v => handleSectionChange("instructions", v)}
        required
        expandedByDefault
      />
      
      {/* Completion Section (optional) */}
      <SectionEditor
        title="Completion"
        content={stepData.sections.completion || ""}
        onChange={v => handleSectionChange("completion", v)}
        collapsible
      />
      
      <button onClick={onSave}>保存</button>
    </div>
  );
}
```

### 3.3 FrontmatterForm Component

```tsx
interface FrontmatterFormProps {
  data: StepFrontmatter;
  agents: AgentListItem[];
  onChange: (field: string, value: any) => void;
}

function FrontmatterForm({ data, agents, onChange }: FrontmatterFormProps) {
  return (
    <div className="frontmatter-form">
      <div className="form-row">
        <label>Node ID</label>
        <input value={data.nodeId} disabled />
      </div>
      
      <div className="form-row">
        <label>Title</label>
        <input 
          value={data.title || ""} 
          onChange={e => onChange("title", e.target.value)} 
        />
      </div>
      
      <div className="form-row">
        <label>Type</label>
        <select value={data.type} onChange={e => onChange("type", e.target.value)}>
          <option value="step">Step</option>
          <option value="decision">Decision</option>
          <option value="merge">Merge</option>
          <option value="end">End</option>
          <option value="subworkflow">Subworkflow</option>
        </select>
      </div>
      
      <div className="form-row">
        <label>Agent</label>
        <select 
          value={data.agentId || ""} 
          onChange={e => onChange("agentId", e.target.value)}
        >
          <option value="">-- Select Agent --</option>
          {agents.map(a => (
            <option key={a.id} value={a.id}>{a.name}</option>
          ))}
        </select>
      </div>
    </div>
  );
}
```

### 3.4 SectionEditor Component

```tsx
interface SectionEditorProps {
  title: string;
  content: string;
  onChange: (value: string) => void;
  required?: boolean;
  collapsible?: boolean;
  expandedByDefault?: boolean;
}

function SectionEditor({ 
  title, 
  content, 
  onChange, 
  required, 
  collapsible,
  expandedByDefault = true 
}: SectionEditorProps) {
  const [expanded, setExpanded] = useState(expandedByDefault);
  
  return (
    <div className="section-editor">
      <div 
        className="section-header" 
        onClick={() => collapsible && setExpanded(!expanded)}
      >
        <h3>
          {title}
          {required && <span className="required">*</span>}
        </h3>
        {collapsible && <span>{expanded ? "▼" : "▶"}</span>}
      </div>
      
      {expanded && (
        <textarea
          value={content}
          onChange={e => onChange(e.target.value)}
          placeholder={`Enter ${title.toLowerCase()}...`}
          rows={title === "Instructions" ? 10 : 4}
        />
      )}
    </div>
  );
}
```

---

## 4. Spec Documentation Update

Add to `spec/bmad-package-spec/v1.1/README.md`:

```markdown
## 7. Step File Structure

### 7.1 Required Format

Every step file MUST follow this structure:

\`\`\`markdown
---
schemaVersion: "1.1"
nodeId: "step-id"
type: "step"
title: "Step Title"
agentId: "agent_id"
---

# Step Title

## Goal

Describe what this step accomplishes.

## Instructions

Detailed instructions for the agent.

## Completion

(Optional) State update rules.
\`\`\`

### 7.2 Frontmatter Fields

| Field | Required | Description |
|:---|:---|:---|
| schemaVersion | ✅ | Must be "1.1" |
| nodeId | ✅ | Unique ID matching graph node |
| type | ✅ | step, decision, merge, end, subworkflow |
| title | ❌ | Display title |
| agentId | ❌ | Agent to execute this step |
| inputs | ❌ | Expected input variables |
| outputs | ❌ | Output variables set by this step |

### 7.3 Required Sections

| Section | Required | Description |
|:---|:---|:---|
| ## Goal | ✅ | What this step accomplishes |
| ## Instructions | ✅ | Detailed agent instructions |
| ## Completion | ❌ | State update rules (Document-as-State) |
```

---

## 5. Test Plan

### Unit Tests

1. `test_validate_step_file_valid` - Valid file passes
2. `test_validate_step_file_no_frontmatter` - Error on missing frontmatter
3. `test_validate_step_file_missing_required_field` - Error on missing nodeId/type
4. `test_validate_step_file_missing_goal` - Error on missing Goal section
5. `test_validate_step_file_missing_instructions` - Error on missing Instructions section

### Integration Tests

1. Import valid package → success
2. Import package with invalid step → detailed error
3. UI displays sections correctly for valid step
4. UI falls back to raw editor for invalid step

---

## 6. Implementation Order

1. **Backend validation** (`validate_step_file`)
2. **Import integration** (call validation)
3. **Spec documentation update**
4. **Update finance-operations examples**
5. **Frontend parser** (`step-parser.ts`)
6. **UI components** (`StepEditorPanel`, `FrontmatterForm`, `SectionEditor`)
7. **Integration** (replace current textarea)
8. **Testing**
