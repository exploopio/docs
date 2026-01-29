# Dataflow Storage Analysis - Phương Án Tốt Nhất

## 1. Đánh Giá Hiện Trạng

### 1.1 EIS Schema (Input Layer)
**File**: `api/pkg/parsers/ris/types.go`

```go
type DataFlow struct {
    Sources       []DataFlowLocation  // Nguồn taint
    Intermediates []DataFlowLocation  // Các bước trung gian
    Sinks         []DataFlowLocation  // Điểm nguy hiểm
    Sanitizers    []DataFlowLocation  // Các bộ lọc
    Tainted       bool                // Còn taint hay không
    TaintType     string              // user_input, file_read, env_var...
    VulnerabilityType string          // sql_injection, xss, ssrf...
    Confidence    int                 // 0-100
    Interprocedural bool              // Cross function
    CrossFile     bool                // Cross file
    CallPath      []string            // Function call chain
    Summary       string              // Human-readable summary
}

type DataFlowLocation struct {
    Path           string   // File path
    Line, EndLine  int      // Line range
    Column, EndColumn int   // Column range
    Content        string   // Code snippet
    Label          string   // Variable name
    Index          int      // Step index
    Type           DataFlowLocationType  // source, sink, propagator, sanitizer
    Function       string   // Function name
    Class          string   // Class name
    Module         string   // Module/namespace
    Operation      string   // assignment, call, return...
    CalledFunction string   // For call operations
    ParameterIndex int      // For parameters
    TaintState     string   // tainted, sanitized, unknown
    Transformation string   // encode, decode, escape...
    Notes          string   // Human-readable notes
}
```

**Điểm mạnh**:
- ✅ Comprehensive: Hỗ trợ đầy đủ taint tracking
- ✅ Tool-agnostic: Có thể nhận data từ Semgrep, CodeQL, Joern, SARIF
- ✅ Helper functions: NewDataFlow, AddIntermediate, BuildSummary...
- ✅ Tested: 13 test cases pass

---

### 1.2 Database Schema (Storage Layer)
**File**: `api/migrations/000127_finding_data_flows.up.sql`

```sql
-- finding_data_flows: Container cho mỗi flow
CREATE TABLE finding_data_flows (
    id UUID PRIMARY KEY,
    finding_id UUID REFERENCES findings(id),
    flow_index INTEGER,     -- Thứ tự trong finding
    message TEXT,           -- Mô tả flow
    importance VARCHAR(20)  -- essential, important, unimportant
);

-- finding_flow_locations: Từng bước trong flow
CREATE TABLE finding_flow_locations (
    id UUID PRIMARY KEY,
    data_flow_id UUID REFERENCES finding_data_flows(id),
    step_index INTEGER,
    location_type VARCHAR(20),  -- source, intermediate, sink, sanitizer

    -- Physical location
    file_path VARCHAR(1000),
    start_line, end_line, start_column, end_column INTEGER,
    snippet TEXT,

    -- Logical location
    function_name, class_name, fully_qualified_name, module_name VARCHAR,

    -- Context
    label VARCHAR(500),
    message TEXT,
    nesting_level INTEGER,
    importance VARCHAR(20)
);
```

**Điểm mạnh**:
- ✅ Normalized: Quan hệ 1-N giữa finding → flows → locations
- ✅ Queryable: Có indexes cho file, function, class, type
- ✅ SARIF compliant: Mapping với codeFlows/threadFlowLocation

**Thiếu sót**:
- ❌ Không có cột `tainted`, `taint_type`, `vulnerability_type`
- ❌ Không có `confidence`, `interprocedural`, `cross_file`
- ❌ Không có `call_path` array
- ❌ Không có `operation`, `called_function`, `taint_state`, `transformation` trong locations

---

### 1.3 Domain Entity (Business Layer)
**File**: `api/internal/domain/vulnerability/data_flow.go`

```go
type FindingDataFlow struct {
    id         shared.ID
    findingID  shared.ID
    flowIndex  int
    message    string
    importance string  // essential, important, unimportant
    createdAt  time.Time
}

type FindingFlowLocation struct {
    id, dataFlowID shared.ID
    stepIndex      int
    locationType   string  // source, intermediate, sink, sanitizer

    // Physical
    filePath, snippet string
    startLine, endLine, startColumn, endColumn int

    // Logical
    functionName, className, fullyQualifiedName, moduleName string

    // Context
    label, message string
    nestingLevel   int
    importance     string
}
```

**Thiếu sót**: Giống database schema

---

## 2. Gap Analysis

### 2.1 EIS → Database Mapping Gaps

| EIS DataFlow Field | Database Column | Status |
|--------------------|-----------------|--------|
| `Sources[]` | `location_type='source'` | ✅ Có thể lưu |
| `Intermediates[]` | `location_type='intermediate'` | ✅ Có thể lưu |
| `Sinks[]` | `location_type='sink'` | ✅ Có thể lưu |
| `Sanitizers[]` | `location_type='sanitizer'` | ✅ Có thể lưu |
| `Tainted` | ❌ **THIẾU** | ⚠️ Cần thêm |
| `TaintType` | ❌ **THIẾU** | ⚠️ Cần thêm |
| `VulnerabilityType` | ❌ **THIẾU** | ⚠️ Cần thêm |
| `Confidence` | ❌ **THIẾU** | ⚠️ Cần thêm |
| `Interprocedural` | ❌ **THIẾU** | ⚠️ Cần thêm |
| `CrossFile` | ❌ **THIẾU** | ⚠️ Cần thêm |
| `CallPath` | ❌ **THIẾU** | ⚠️ Cần thêm |
| `Summary` | `message` | ✅ Mapping hiện có |

| EIS DataFlowLocation Field | Database Column | Status |
|---------------------------|-----------------|--------|
| `Path` | `file_path` | ✅ |
| `Line`, `EndLine` | `start_line`, `end_line` | ✅ |
| `Column`, `EndColumn` | `start_column`, `end_column` | ✅ |
| `Content` | `snippet` | ✅ |
| `Label` | `label` | ✅ |
| `Index` | `step_index` | ✅ |
| `Type` | `location_type` | ✅ |
| `Function` | `function_name` | ✅ |
| `Class` | `class_name` | ✅ |
| `Module` | `module_name` | ✅ |
| `Operation` | ❌ **THIẾU** | ⚠️ Cần thêm |
| `CalledFunction` | ❌ **THIẾU** | ⚠️ Cần thêm |
| `ParameterIndex` | ❌ **THIẾU** | ⚠️ Cần thêm |
| `TaintState` | ❌ **THIẾU** | ⚠️ Cần thêm |
| `Transformation` | ❌ **THIẾU** | ⚠️ Cần thêm |
| `Notes` | `message` | ✅ (có thể dùng) |

---

## 3. Phương Án Đề Xuất

### Phương Án A: Extend Database Schema (Recommended)

**Ưu điểm**: Full queryability, strong typing, native SQL queries
**Nhược điểm**: Migration required

#### Migration 000129: Extend Data Flow Tables

```sql
-- Add missing columns to finding_data_flows
ALTER TABLE finding_data_flows ADD COLUMN IF NOT EXISTS tainted BOOLEAN DEFAULT true;
ALTER TABLE finding_data_flows ADD COLUMN IF NOT EXISTS taint_type VARCHAR(50);
ALTER TABLE finding_data_flows ADD COLUMN IF NOT EXISTS vulnerability_type VARCHAR(50);
ALTER TABLE finding_data_flows ADD COLUMN IF NOT EXISTS confidence INTEGER;
ALTER TABLE finding_data_flows ADD COLUMN IF NOT EXISTS interprocedural BOOLEAN DEFAULT false;
ALTER TABLE finding_data_flows ADD COLUMN IF NOT EXISTS cross_file BOOLEAN DEFAULT false;
ALTER TABLE finding_data_flows ADD COLUMN IF NOT EXISTS call_path TEXT[];  -- PostgreSQL array

-- Add indexes
CREATE INDEX IF NOT EXISTS idx_data_flows_taint_type ON finding_data_flows(taint_type) WHERE taint_type IS NOT NULL;
CREATE INDEX IF NOT EXISTS idx_data_flows_vuln_type ON finding_data_flows(vulnerability_type) WHERE vulnerability_type IS NOT NULL;
CREATE INDEX IF NOT EXISTS idx_data_flows_interprocedural ON finding_data_flows(interprocedural) WHERE interprocedural = true;
CREATE INDEX IF NOT EXISTS idx_data_flows_cross_file ON finding_data_flows(cross_file) WHERE cross_file = true;

-- Add missing columns to finding_flow_locations
ALTER TABLE finding_flow_locations ADD COLUMN IF NOT EXISTS operation VARCHAR(50);
ALTER TABLE finding_flow_locations ADD COLUMN IF NOT EXISTS called_function VARCHAR(500);
ALTER TABLE finding_flow_locations ADD COLUMN IF NOT EXISTS parameter_index INTEGER;
ALTER TABLE finding_flow_locations ADD COLUMN IF NOT EXISTS taint_state VARCHAR(20);
ALTER TABLE finding_flow_locations ADD COLUMN IF NOT EXISTS transformation VARCHAR(50);

-- Add indexes
CREATE INDEX IF NOT EXISTS idx_flow_locations_operation ON finding_flow_locations(operation) WHERE operation IS NOT NULL;
CREATE INDEX IF NOT EXISTS idx_flow_locations_taint_state ON finding_flow_locations(taint_state) WHERE taint_state IS NOT NULL;
```

---

### Phương Án B: Hybrid (JSONB Fallback)

**Ưu điểm**: No migration, flexible
**Nhược điểm**: Less queryable, duplicated data

```sql
-- Add JSONB column for extended metadata
ALTER TABLE finding_data_flows ADD COLUMN IF NOT EXISTS extended_metadata JSONB;
ALTER TABLE finding_flow_locations ADD COLUMN IF NOT EXISTS extended_metadata JSONB;
```

Lưu các field mới vào JSONB:
```json
{
  "tainted": true,
  "taint_type": "user_input",
  "vulnerability_type": "sql_injection",
  "confidence": 95,
  "interprocedural": true,
  "cross_file": false,
  "call_path": ["getInput", "processQuery", "executeSQL"]
}
```

---

### Phương Án C: Keep Current (Minimal)

**Ưu điểm**: No changes
**Nhược điểm**: Information loss, limited analysis

Chỉ lưu các trường hiện có, bỏ qua các trường mở rộng. Không khuyến khích.

---

## 4. Khuyến Nghị: Phương Án A + Conversion Layer

### 4.1 Database Migration

```sql
-- api/migrations/000129_extend_data_flow_tables.up.sql
-- (Như trên)
```

### 4.2 Domain Entity Updates

```go
// api/internal/domain/vulnerability/data_flow.go

type FindingDataFlow struct {
    // ... existing fields ...

    // NEW: Extended taint tracking fields
    tainted         bool
    taintType       string   // user_input, file_read, env_var, network
    vulnerabilityType string // sql_injection, xss, ssrf, command_injection
    confidence      int      // 0-100
    interprocedural bool
    crossFile       bool
    callPath        []string // Function call chain
}

type FindingFlowLocation struct {
    // ... existing fields ...

    // NEW: Extended operation tracking
    operation      string // assignment, call, return, concat
    calledFunction string // For call operations
    parameterIndex int    // For parameter tracking
    taintState     string // tainted, sanitized, unknown
    transformation string // encode, decode, escape, hash
}
```

### 4.3 EIS → Domain Converter

```go
// api/internal/app/ingest/dataflow_converter.go

func ConvertEISDataFlowToFindingDataFlows(
    findingID shared.ID,
    risDataFlow *ris.DataFlow,
) ([]*vulnerability.FindingDataFlow, []*vulnerability.FindingFlowLocation, error) {
    if risDataFlow == nil {
        return nil, nil, nil
    }

    // Create single FindingDataFlow from EIS DataFlow
    flow, err := vulnerability.NewFindingDataFlow(findingID, 0, risDataFlow.Summary, "essential")
    if err != nil {
        return nil, nil, err
    }

    // Set extended fields
    flow.SetTainted(risDataFlow.Tainted)
    flow.SetTaintType(risDataFlow.TaintType)
    flow.SetVulnerabilityType(risDataFlow.VulnerabilityType)
    flow.SetConfidence(risDataFlow.Confidence)
    flow.SetInterprocedural(risDataFlow.Interprocedural)
    flow.SetCrossFile(risDataFlow.CrossFile)
    flow.SetCallPath(risDataFlow.CallPath)

    var locations []*vulnerability.FindingFlowLocation
    stepIndex := 0

    // Convert sources
    for _, src := range risDataFlow.Sources {
        loc := convertEISLocationToDomain(flow.ID(), stepIndex, vulnerability.LocationTypeSource, src)
        locations = append(locations, loc)
        stepIndex++
    }

    // Convert intermediates
    for _, inter := range risDataFlow.Intermediates {
        loc := convertEISLocationToDomain(flow.ID(), stepIndex, vulnerability.LocationTypeIntermediate, inter)
        locations = append(locations, loc)
        stepIndex++
    }

    // Convert sanitizers
    for _, san := range risDataFlow.Sanitizers {
        loc := convertEISLocationToDomain(flow.ID(), stepIndex, vulnerability.LocationTypeSanitizer, san)
        locations = append(locations, loc)
        stepIndex++
    }

    // Convert sinks
    for _, sink := range risDataFlow.Sinks {
        loc := convertEISLocationToDomain(flow.ID(), stepIndex, vulnerability.LocationTypeSink, sink)
        locations = append(locations, loc)
        stepIndex++
    }

    return []*vulnerability.FindingDataFlow{flow}, locations, nil
}

func convertEISLocationToDomain(
    dataFlowID shared.ID,
    stepIndex int,
    locType string,
    risLoc ris.DataFlowLocation,
) *vulnerability.FindingFlowLocation {
    loc, _ := vulnerability.NewFindingFlowLocation(dataFlowID, stepIndex, locType)

    // Physical
    loc.SetPhysicalLocation(
        risLoc.Path,
        risLoc.Line,
        risLoc.EndLine,
        risLoc.Column,
        risLoc.EndColumn,
        risLoc.Content,
    )

    // Logical
    loc.SetLogicalLocation(
        risLoc.Function,
        risLoc.Class,
        "", // FQN computed
        risLoc.Module,
    )

    // Context
    loc.SetContext(risLoc.Label, risLoc.Notes, 0, "essential")

    // Extended fields
    loc.SetOperation(risLoc.Operation)
    loc.SetCalledFunction(risLoc.CalledFunction)
    loc.SetParameterIndex(risLoc.ParameterIndex)
    loc.SetTaintState(risLoc.TaintState)
    loc.SetTransformation(risLoc.Transformation)

    return loc
}
```

---

## 5. Tool Compatibility Matrix

| Tool | Output Format | EIS Mapping | DB Storage |
|------|--------------|-------------|------------|
| **Semgrep OSS** | SARIF | `codeFlows` → `DataFlow` | ✅ Full |
| **Semgrep Pro** | SARIF + Extended | Full taint info | ✅ Full |
| **CodeQL** | SARIF | `codeFlows` → `DataFlow` | ✅ Full |
| **Joern** | Custom JSON | Convert to `DataFlow` | ✅ Full |
| **Snyk Code** | SARIF | `codeFlows` → `DataFlow` | ✅ Full |
| **SonarQube** | API JSON | Convert to `DataFlow` | ✅ Full |
| **Checkmarx** | XML/JSON | Convert to `DataFlow` | ✅ Full |
| **Bandit** | SARIF/JSON | Limited flow info | ⚠️ Partial |
| **Gosec** | SARIF/JSON | Limited flow info | ⚠️ Partial |

---

## 6. Query Examples

### 6.1 Find all SQL injection dataflows
```sql
SELECT
    f.id as finding_id,
    f.title,
    df.vulnerability_type,
    df.taint_type,
    df.confidence
FROM findings f
JOIN finding_data_flows df ON df.finding_id = f.id
WHERE df.vulnerability_type = 'sql_injection'
  AND df.tainted = true
ORDER BY df.confidence DESC;
```

### 6.2 Find cross-file attack paths
```sql
SELECT
    df.id,
    df.call_path,
    array_agg(DISTINCT fl.file_path ORDER BY fl.step_index) as files_traversed
FROM finding_data_flows df
JOIN finding_flow_locations fl ON fl.data_flow_id = df.id
WHERE df.cross_file = true
GROUP BY df.id, df.call_path
HAVING count(DISTINCT fl.file_path) > 1;
```

### 6.3 Find all sources in a specific file
```sql
SELECT
    fl.*,
    df.taint_type
FROM finding_flow_locations fl
JOIN finding_data_flows df ON df.id = fl.data_flow_id
WHERE fl.file_path LIKE '%controller.php'
  AND fl.location_type = 'source';
```

### 6.4 Find functions that act as propagators
```sql
SELECT
    function_name,
    count(*) as propagation_count,
    array_agg(DISTINCT file_path) as files
FROM finding_flow_locations
WHERE location_type = 'intermediate'
  AND function_name IS NOT NULL
GROUP BY function_name
ORDER BY propagation_count DESC
LIMIT 20;
```

---

## 7. Implementation Plan

### Phase 1: Database Migration ⏱️ 1-2 hours
1. Create `000129_extend_data_flow_tables.up.sql`
2. Create `000129_extend_data_flow_tables.down.sql`
3. Run on staging

### Phase 2: Domain Layer ⏱️ 2-3 hours
1. Update `FindingDataFlow` struct with new fields
2. Update `FindingFlowLocation` struct with new fields
3. Add setters and reconstitute functions
4. Update `DataFlowRepository` interface

### Phase 3: Repository Layer ⏱️ 2-3 hours
1. Update `data_flow_repository.go` to handle new columns
2. Update SQL queries for insert/update/select
3. Add new query methods for analytics

### Phase 4: Ingestion Pipeline ⏱️ 2-3 hours
1. Create `dataflow_converter.go`
2. Update `processor_findings.go` to use converter
3. Handle backward compatibility

### Phase 5: Testing ⏱️ 1-2 hours
1. Unit tests for converter
2. Integration tests for repository
3. E2E test with actual tool output

---

## 8. Conclusion

**Khuyến nghị**: Thực hiện **Phương Án A** với migration mở rộng database schema.

**Lý do**:
1. ✅ Full queryability cho tất cả trường mới
2. ✅ Native SQL support cho analytics và reporting
3. ✅ Strong typing với CHECK constraints
4. ✅ Indexes cho performance
5. ✅ Backward compatible (NULL allowed cho các cột mới)
6. ✅ Tool-agnostic: Bất kỳ tool nào output dataflow đều lưu được

**Rủi ro**: Low - Chỉ ADD columns, không thay đổi existing data.
