# Felorx Sentry MVP Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add `felorx-sentry` as an independent submodule and refactor it into a Sentry-compatible MVP with project management, official payload preservation, issue aggregation, logs, traces, and Felorx OIDC-backed admin access.

**Architecture:** Keep `felorx-sentry` independent under `apps/sentry`, with Go + Gin + GORM + PostgreSQL for API/ingest and Next.js for the admin UI. Split the Go API into `auth`, `projects`, `ingest`, `events`, `issues`, `logs`, `traces`, and `shared` packages; preserve official Sentry payloads as JSONB while adding narrow index columns for search and aggregation.

**Tech Stack:** Git submodules, Go 1.23+, Gin, GORM, PostgreSQL, SQLite test harness, Next.js 15, React 19, TypeScript, Vitest, Felorx OIDC.

---

## Scope Check

This plan covers one MVP release but spans multiple surfaces. The work is ordered so each task leaves the system buildable or locally testable. If execution is split across agents, do not run tasks that modify the same files concurrently; Task 1 must land before all `apps/sentry` tasks.

## File Structure

Root repository:

- Modify: `.gitmodules`
- Create: `apps/sentry` submodule pointer

Inside `apps/sentry`:

- Modify: `go.mod`
- Modify: `cmd/api/main.go`
- Modify: `internal/config/config.go`
- Delete after migration: `internal/api/handler.go`
- Delete after migration: `internal/api/dto.go`
- Delete after migration: `internal/api/sentry.go`
- Delete after migration: `internal/database/database.go`
- Create: `internal/shared/response.go`
- Create: `internal/shared/time.go`
- Create: `internal/shared/json.go`
- Create: `internal/storage/models.go`
- Create: `internal/storage/database.go`
- Create: `internal/auth/authenticator.go`
- Create: `internal/auth/dev_authenticator.go`
- Create: `internal/auth/oidc_authenticator.go`
- Create: `internal/auth/middleware.go`
- Create: `internal/projects/service.go`
- Create: `internal/projects/handler.go`
- Create: `internal/ingest/envelope.go`
- Create: `internal/ingest/dsn.go`
- Create: `internal/ingest/extractors.go`
- Create: `internal/ingest/handler.go`
- Create: `internal/issues/grouping.go`
- Create: `internal/issues/service.go`
- Create: `internal/events/service.go`
- Create: `internal/events/handler.go`
- Create: `internal/logs/service.go`
- Create: `internal/logs/handler.go`
- Create: `internal/traces/service.go`
- Create: `internal/traces/handler.go`
- Create: `internal/router/router.go`
- Create: `internal/testutil/db.go`
- Create: `internal/testutil/fixtures.go`
- Create: Go tests alongside every new package listed above.

Inside `apps/sentry/web`:

- Modify: `package.json`
- Modify: `app/layout.tsx`
- Replace: `app/page.tsx`
- Replace: `app/events/[id]/page.tsx`
- Create: `app/login/page.tsx`
- Create: `app/projects/page.tsx`
- Create: `app/projects/[projectId]/page.tsx`
- Create: `app/projects/[projectId]/settings/page.tsx`
- Create: `app/projects/[projectId]/issues/page.tsx`
- Create: `app/projects/[projectId]/issues/[issueId]/page.tsx`
- Create: `app/projects/[projectId]/events/[eventId]/page.tsx`
- Create: `app/projects/[projectId]/logs/page.tsx`
- Create: `app/projects/[projectId]/traces/page.tsx`
- Create: `app/projects/[projectId]/traces/[traceId]/page.tsx`
- Create: `app/api/auth/[...nextauth]/route.ts`
- Create: `lib/api/client.ts`
- Create: `lib/api/types.ts`
- Create: `lib/auth/options.ts`
- Create: `components/AppShell.tsx`
- Create: `components/JsonPanel.tsx`
- Create: `components/StatusBadge.tsx`
- Create: `components/TraceTree.tsx`
- Create: `components/__tests__/StatusBadge.test.tsx`
- Create: `components/__tests__/TraceTree.test.tsx`
- Create: `lib/api/__tests__/client.test.ts`
- Create: `vitest.config.ts`
- Create: `test/setup.ts`

Docs and deployment inside `apps/sentry`:

- Modify: `README.md`
- Modify: `docker-compose.yml`
- Modify: `Dockerfile`
- Modify: `web/Dockerfile`
- Create: `.env.example`

---

## Task 1: Add `felorx-sentry` Submodule And Baseline

**Files:**
- Modify: `.gitmodules`
- Create: `apps/sentry`
- Modify: `apps/sentry/go.mod`
- Modify: imports under `apps/sentry/cmd` and `apps/sentry/internal`

- [ ] **Step 1: Add the submodule**

Run:

```bash
git submodule add -b main git@github.com:felorx/felorx-sentry.git apps/sentry
```

Expected:

```text
Cloning into 'apps/sentry'...
```

- [ ] **Step 2: Verify submodule metadata**

Run:

```bash
git config --file .gitmodules --get submodule.apps/sentry.path
git config --file .gitmodules --get submodule.apps/sentry.url
git -C apps/sentry status --short --branch
```

Expected:

```text
apps/sentry
git@github.com:felorx/felorx-sentry.git
## main...origin/main
```

- [ ] **Step 3: Change Go module path**

Modify `apps/sentry/go.mod`:

```go
module github.com/felorx/felorx-sentry
```

Replace imports:

```bash
cd apps/sentry
gofmt -w cmd internal
go mod tidy
```

Expected import namespace after replacement:

```text
github.com/felorx/felorx-sentry/internal/config
github.com/felorx/felorx-sentry/internal/storage
github.com/felorx/felorx-sentry/internal/ingest
```

- [ ] **Step 4: Run baseline builds**

Run:

```bash
cd apps/sentry
go test ./...
cd web
npm install
npm run build
```

Expected: Go tests pass or report no test files; Next.js build succeeds with the current simple UI.

- [ ] **Step 5: Commit root and submodule baseline**

Inside submodule:

```bash
cd apps/sentry
git add go.mod go.sum cmd internal
git commit -m "chore: 迁移 Felorx Sentry 模块路径"
```

In root:

```bash
git add .gitmodules apps/sentry
git commit -m "chore(sentry): 添加 Felorx Sentry 子模块"
```

---

## Task 2: Replace Envelope Parser With Official-Length Parser

**Files:**
- Create: `apps/sentry/internal/ingest/envelope.go`
- Create: `apps/sentry/internal/ingest/envelope_test.go`
- Remove after route migration: `apps/sentry/internal/api/sentry.go`

- [ ] **Step 1: Write failing parser tests**

Create `apps/sentry/internal/ingest/envelope_test.go`:

```go
package ingest

import (
	"bytes"
	"compress/gzip"
	"testing"
)

func TestParseEnvelopeReadsLengthPrefixedPayloadWithNewline(t *testing.T) {
	body := []byte("{\"event_id\":\"fc6d8c0c43fc4630ad850ee518f1b9d0\"}\n" +
		"{\"type\":\"event\",\"length\":55}\n" +
		"{\"message\":\"first line\\nsecond line\",\"level\":\"error\"}\n")

	envelope, err := ParseEnvelope(bytes.NewReader(body), "")
	if err != nil {
		t.Fatalf("ParseEnvelope returned error: %v", err)
	}

	if got := envelope.Header["event_id"]; got != "fc6d8c0c43fc4630ad850ee518f1b9d0" {
		t.Fatalf("event_id = %v", got)
	}
	if len(envelope.Items) != 1 {
		t.Fatalf("items = %d", len(envelope.Items))
	}
	if string(envelope.Items[0].Payload) != "{\"message\":\"first line\\nsecond line\",\"level\":\"error\"}" {
		t.Fatalf("payload = %q", string(envelope.Items[0].Payload))
	}
}

func TestParseEnvelopeRetainsUnknownItem(t *testing.T) {
	body := []byte("{}\n{\"type\":\"profile\",\"length\":11}\nraw-profile\n")

	envelope, err := ParseEnvelope(bytes.NewReader(body), "")
	if err != nil {
		t.Fatalf("ParseEnvelope returned error: %v", err)
	}

	if envelope.Items[0].Type != "profile" {
		t.Fatalf("type = %s", envelope.Items[0].Type)
	}
	if string(envelope.Items[0].Payload) != "raw-profile" {
		t.Fatalf("payload = %q", string(envelope.Items[0].Payload))
	}
}

func TestParseEnvelopeHandlesGzipContentEncoding(t *testing.T) {
	var compressed bytes.Buffer
	gz := gzip.NewWriter(&compressed)
	_, _ = gz.Write([]byte("{}\n{\"type\":\"event\"}\n{\"event_id\":\"fc6d8c0c43fc4630ad850ee518f1b9d0\"}\n"))
	_ = gz.Close()

	envelope, err := ParseEnvelope(bytes.NewReader(compressed.Bytes()), "gzip")
	if err != nil {
		t.Fatalf("ParseEnvelope returned error: %v", err)
	}

	if len(envelope.Items) != 1 {
		t.Fatalf("items = %d", len(envelope.Items))
	}
}
```

- [ ] **Step 2: Run tests and verify RED**

Run:

```bash
cd apps/sentry
go test ./internal/ingest -run TestParseEnvelope
```

Expected: FAIL because `ParseEnvelope`, `Envelope`, and `EnvelopeItem` do not exist in the new package.

- [ ] **Step 3: Implement parser API**

Create `apps/sentry/internal/ingest/envelope.go` with this public API:

```go
package ingest

import (
	"bufio"
	"bytes"
	"compress/gzip"
	"encoding/json"
	"fmt"
	"io"
)

type Envelope struct {
	Header map[string]any
	Items  []EnvelopeItem
}

type EnvelopeItem struct {
	Type        string
	ContentType string
	ItemCount   int
	Header      map[string]any
	Payload     []byte
}

func ParseEnvelope(reader io.Reader, contentEncoding string) (*Envelope, error) {
	data, err := io.ReadAll(reader)
	if err != nil {
		return nil, fmt.Errorf("read envelope: %w", err)
	}
	if contentEncoding == "gzip" || hasGzipMagic(data) {
		gz, err := gzip.NewReader(bytes.NewReader(data))
		if err != nil {
			return nil, fmt.Errorf("open gzip envelope: %w", err)
		}
		defer gz.Close()
		data, err = io.ReadAll(gz)
		if err != nil {
			return nil, fmt.Errorf("read gzip envelope: %w", err)
		}
	}

	scanner := bufio.NewReader(bytes.NewReader(data))
	headerLine, err := readHeaderLine(scanner)
	if err != nil {
		return nil, err
	}
	var header map[string]any
	if err := json.Unmarshal(headerLine, &header); err != nil {
		return nil, fmt.Errorf("parse envelope header: %w", err)
	}

	envelope := &Envelope{Header: header}
	for {
		itemHeaderLine, err := readHeaderLine(scanner)
		if err == io.EOF {
			return envelope, nil
		}
		if err != nil {
			return nil, err
		}
		if len(bytes.TrimSpace(itemHeaderLine)) == 0 {
			continue
		}

		var itemHeader map[string]any
		if err := json.Unmarshal(itemHeaderLine, &itemHeader); err != nil {
			return nil, fmt.Errorf("parse item header: %w", err)
		}

		payload, err := readPayload(scanner, itemHeader)
		if err != nil {
			return nil, err
		}
		envelope.Items = append(envelope.Items, EnvelopeItem{
			Type:        stringValue(itemHeader, "type"),
			ContentType: stringValue(itemHeader, "content_type"),
			ItemCount:   intValue(itemHeader, "item_count"),
			Header:      itemHeader,
			Payload:     payload,
		})
	}
}
```

Implement private helpers in the same file:

```go
func hasGzipMagic(data []byte) bool {
	return len(data) >= 2 && data[0] == 0x1f && data[1] == 0x8b
}

func readHeaderLine(reader *bufio.Reader) ([]byte, error) {
	line, err := reader.ReadBytes('\n')
	if err != nil {
		return nil, err
	}
	return bytes.TrimSuffix(line, []byte{'\n'}), nil
}

func readPayload(reader *bufio.Reader, header map[string]any) ([]byte, error) {
	length := intValue(header, "length")
	if length > 0 {
		payload := make([]byte, length)
		if _, err := io.ReadFull(reader, payload); err != nil {
			return nil, fmt.Errorf("read length-prefixed payload: %w", err)
		}
		next, err := reader.Peek(1)
		if err == nil && len(next) == 1 && next[0] == '\n' {
			_, _ = reader.ReadByte()
		}
		return payload, nil
	}

	line, err := reader.ReadBytes('\n')
	if err != nil && err != io.EOF {
		return nil, fmt.Errorf("read newline payload: %w", err)
	}
	return bytes.TrimSuffix(line, []byte{'\n'}), nil
}

func stringValue(data map[string]any, key string) string {
	value, _ := data[key].(string)
	return value
}

func intValue(data map[string]any, key string) int {
	switch value := data[key].(type) {
	case int:
		return value
	case int64:
		return int(value)
	case float64:
		return int(value)
	default:
		return 0
	}
}
```

`readPayload` must read exactly `length` bytes when the header has `length`; otherwise it reads one newline-terminated payload line. When `length` is present, consume the following newline if present and do not include it in `Payload`.

- [ ] **Step 4: Run parser tests and gofmt**

Run:

```bash
cd apps/sentry
gofmt -w internal/ingest/envelope.go internal/ingest/envelope_test.go
go test ./internal/ingest -run TestParseEnvelope
```

Expected: all parser tests pass.

- [ ] **Step 5: Commit**

```bash
cd apps/sentry
git add internal/ingest/envelope.go internal/ingest/envelope_test.go
git commit -m "feat(ingest): 支持官方 Sentry envelope 解析"
```

---

## Task 3: Add Storage Models And Migration Harness

**Files:**
- Create: `apps/sentry/internal/storage/models.go`
- Create: `apps/sentry/internal/storage/database.go`
- Create: `apps/sentry/internal/storage/models_test.go`
- Create: `apps/sentry/internal/testutil/db.go`
- Modify: `apps/sentry/go.mod`

- [ ] **Step 1: Write failing model migration test**

Create `apps/sentry/internal/storage/models_test.go`:

```go
package storage

import (
	"testing"

	"github.com/felorx/felorx-sentry/internal/testutil"
)

func TestAutoMigrateCreatesMvpTables(t *testing.T) {
	db := testutil.OpenSQLite(t)
	if err := AutoMigrate(db); err != nil {
		t.Fatalf("AutoMigrate returned error: %v", err)
	}

	for _, table := range []string{
		"projects",
		"project_members",
		"envelopes",
		"envelope_items",
		"events",
		"issues",
		"logs",
		"traces",
		"spans",
	} {
		if !db.Migrator().HasTable(table) {
			t.Fatalf("missing table %s", table)
		}
	}
}
```

Create `apps/sentry/internal/testutil/db.go`:

```go
package testutil

import (
	"testing"

	"gorm.io/driver/sqlite"
	"gorm.io/gorm"
)

func OpenSQLite(t *testing.T) *gorm.DB {
	t.Helper()
	db, err := gorm.Open(sqlite.Open(":memory:"), &gorm.Config{})
	if err != nil {
		t.Fatalf("open sqlite: %v", err)
	}
	return db
}
```

- [ ] **Step 2: Run tests and verify RED**

Run:

```bash
cd apps/sentry
go get gorm.io/driver/sqlite gorm.io/datatypes github.com/google/uuid
go test ./internal/storage -run TestAutoMigrateCreatesMvpTables
```

Expected: FAIL because `AutoMigrate` and models are not implemented.

- [ ] **Step 3: Implement MVP models**

Create `apps/sentry/internal/storage/models.go` with these GORM models:

```go
package storage

import (
	"time"

	"github.com/google/uuid"
	"gorm.io/datatypes"
	"gorm.io/gorm"
)

type BaseModel struct {
	ID        uuid.UUID `gorm:"type:uuid;primaryKey"`
	CreatedAt time.Time
	UpdatedAt time.Time
}

func (m *BaseModel) BeforeCreate(tx *gorm.DB) error {
	if m.ID == uuid.Nil {
		m.ID = uuid.New()
	}
	return nil
}

type Project struct {
	BaseModel
	Slug                  string `gorm:"uniqueIndex;size:120;not null"`
	Name                  string `gorm:"size:200;not null"`
	Platform              string `gorm:"size:80"`
	Status                string `gorm:"size:40;index;not null;default:active"`
	DsnPublicKey          string `gorm:"uniqueIndex;size:80;not null"`
	DsnSecretKeyHash      string `gorm:"size:160"`
	FelorxAppID           string `gorm:"size:120;index"`
	FelorxAppName         string `gorm:"size:200"`
	CreatedByFelorxUserID string `gorm:"size:120;index;not null"`
}

type ProjectMember struct {
	BaseModel
	ProjectID    uuid.UUID `gorm:"type:uuid;index;not null"`
	FelorxUserID string    `gorm:"size:120;index;not null"`
	Email        string    `gorm:"size:200"`
	Role         string    `gorm:"size:40;not null"`
}

type Envelope struct {
	BaseModel
	ProjectID  uuid.UUID      `gorm:"type:uuid;index;not null"`
	EventID    string         `gorm:"size:64;index"`
	DSN        string         `gorm:"column:dsn;size:500"`
	SDK        datatypes.JSON `gorm:"type:jsonb"`
	SentAt     *time.Time
	Headers    datatypes.JSON `gorm:"type:jsonb;not null"`
	ReceivedAt time.Time      `gorm:"index;not null"`
	RemoteAddr string         `gorm:"size:120"`
	UserAgent  string         `gorm:"size:500"`
}

type EnvelopeItem struct {
	BaseModel
	EnvelopeID  *uuid.UUID     `gorm:"type:uuid;index"`
	ProjectID   uuid.UUID      `gorm:"type:uuid;index;not null"`
	ItemType    string         `gorm:"size:80;index;not null"`
	ContentType string         `gorm:"size:160"`
	ItemCount   int
	Headers     datatypes.JSON `gorm:"type:jsonb;not null"`
	Payload     datatypes.JSON `gorm:"type:jsonb"`
	PayloadText string         `gorm:"type:text"`
	PayloadBytes []byte
	PayloadSize int64
}

type Event struct {
	BaseModel
	ProjectID     uuid.UUID      `gorm:"type:uuid;index;not null"`
	IssueID       *uuid.UUID     `gorm:"type:uuid;index"`
	EnvelopeItemID *uuid.UUID    `gorm:"type:uuid;index"`
	EventID       string         `gorm:"size:64;index;not null"`
	Type          string         `gorm:"size:40;index;not null"`
	Timestamp     time.Time      `gorm:"index;not null"`
	Platform      string         `gorm:"size:80"`
	Level         string         `gorm:"size:40;index"`
	Logger        string         `gorm:"size:200;index"`
	Message       string         `gorm:"type:text"`
	Transaction   string         `gorm:"size:500;index"`
	ServerName    string         `gorm:"size:200"`
	Release       string         `gorm:"size:200;index"`
	Dist          string         `gorm:"size:120"`
	Environment   string        `gorm:"size:120;index"`
	Fingerprint   string         `gorm:"size:300;index"`
	ExceptionType string         `gorm:"size:200;index"`
	TraceID       string         `gorm:"size:64;index"`
	SpanID        string         `gorm:"size:32;index"`
	UserID        string         `gorm:"size:200;index"`
	RawPayload    datatypes.JSON `gorm:"type:jsonb;not null"`
}

type Issue struct {
	BaseModel
	ProjectID     uuid.UUID      `gorm:"type:uuid;uniqueIndex:idx_issue_fingerprint;not null"`
	Fingerprint   string         `gorm:"size:300;uniqueIndex:idx_issue_fingerprint;not null"`
	Title         string         `gorm:"size:500;not null"`
	Culprit       string         `gorm:"size:500"`
	ExceptionType string         `gorm:"size:200;index"`
	Level         string         `gorm:"size:40;index"`
	Status        string         `gorm:"size:40;index;not null"`
	EventCount    int64          `gorm:"not null"`
	FirstSeen     time.Time      `gorm:"index;not null"`
	LastSeen      time.Time      `gorm:"index;not null"`
	LastEventID   *uuid.UUID     `gorm:"type:uuid;index"`
	Metadata      datatypes.JSON `gorm:"type:jsonb"`
}

type Log struct {
	BaseModel
	ProjectID      uuid.UUID      `gorm:"type:uuid;index;not null"`
	EnvelopeItemID *uuid.UUID     `gorm:"type:uuid;index"`
	EventID        *uuid.UUID     `gorm:"type:uuid;index"`
	Timestamp      time.Time      `gorm:"index;not null"`
	TraceID        string         `gorm:"size:64;index"`
	SpanID         string         `gorm:"size:32;index"`
	Level          string         `gorm:"size:40;index;not null"`
	Body           string         `gorm:"type:text;not null"`
	SeverityNumber int            `gorm:"index"`
	Attributes     datatypes.JSON `gorm:"type:jsonb"`
	RawPayload     datatypes.JSON `gorm:"type:jsonb;not null"`
}

type Trace struct {
	BaseModel
	ProjectID      uuid.UUID  `gorm:"type:uuid;uniqueIndex:idx_trace_project;not null"`
	TraceID        string     `gorm:"size:64;uniqueIndex:idx_trace_project;not null"`
	Name           string     `gorm:"size:500"`
	Operation      string     `gorm:"size:120;index"`
	Status         string     `gorm:"size:80;index"`
	StartTimestamp *time.Time `gorm:"index"`
	EndTimestamp   *time.Time `gorm:"index"`
	DurationMS     float64
	RootEventID    *uuid.UUID `gorm:"type:uuid;index"`
}

type Span struct {
	BaseModel
	ProjectID      uuid.UUID      `gorm:"type:uuid;uniqueIndex:idx_span_identity;not null"`
	TraceID        string         `gorm:"size:64;uniqueIndex:idx_span_identity;not null"`
	SpanID         string         `gorm:"size:32;uniqueIndex:idx_span_identity;not null"`
	ParentSpanID   string         `gorm:"size:32;index"`
	Name           string         `gorm:"size:500;not null"`
	Operation      string         `gorm:"size:120;index"`
	Status         string         `gorm:"size:80;index"`
	IsSegment      bool           `gorm:"index"`
	StartTimestamp time.Time      `gorm:"index;not null"`
	EndTimestamp   time.Time      `gorm:"index;not null"`
	DurationMS     float64
	Attributes     datatypes.JSON `gorm:"type:jsonb"`
	Links          datatypes.JSON `gorm:"type:jsonb"`
	RawPayload     datatypes.JSON `gorm:"type:jsonb;not null"`
}
```

Use `datatypes.JSON` for JSONB-capable fields and `uuid.UUID` for IDs. Use `BeforeCreate` hooks or a shared helper so zero UUIDs are generated before insert.

Create `apps/sentry/internal/storage/database.go`:

```go
package storage

import "gorm.io/gorm"

func AutoMigrate(db *gorm.DB) error {
	return db.AutoMigrate(
		&Project{},
		&ProjectMember{},
		&Envelope{},
		&EnvelopeItem{},
		&Event{},
		&Issue{},
		&Log{},
		&Trace{},
		&Span{},
	)
}
```

- [ ] **Step 4: Run model tests**

Run:

```bash
cd apps/sentry
gofmt -w internal/storage internal/testutil
go test ./internal/storage
```

Expected: tests pass.

- [ ] **Step 5: Commit**

```bash
cd apps/sentry
git add go.mod go.sum internal/storage internal/testutil
git commit -m "feat(storage): 增加 Sentry MVP 数据模型"
```

---

## Task 4: Implement Projects And DSN Management

**Files:**
- Create: `apps/sentry/internal/projects/service.go`
- Create: `apps/sentry/internal/projects/service_test.go`
- Create: `apps/sentry/internal/ingest/dsn.go`
- Create: `apps/sentry/internal/ingest/dsn_test.go`

- [ ] **Step 1: Write failing DSN tests**

Create `apps/sentry/internal/ingest/dsn_test.go`:

```go
package ingest

import "testing"

func TestParseDSNExtractsPublicKeyAndProjectSlug(t *testing.T) {
	parsed, err := ParseDSN("https://public-key@sentry.felorx.com/my-app")
	if err != nil {
		t.Fatalf("ParseDSN returned error: %v", err)
	}
	if parsed.PublicKey != "public-key" {
		t.Fatalf("PublicKey = %s", parsed.PublicKey)
	}
	if parsed.ProjectID != "my-app" {
		t.Fatalf("ProjectID = %s", parsed.ProjectID)
	}
}
```

- [ ] **Step 2: Write failing project service tests**

Create `apps/sentry/internal/projects/service_test.go`:

```go
package projects

import (
	"testing"

	"github.com/felorx/felorx-sentry/internal/storage"
	"github.com/felorx/felorx-sentry/internal/testutil"
)

func TestCreateProjectCreatesOwnerAndDSN(t *testing.T) {
	db := testutil.OpenSQLite(t)
	if err := storage.AutoMigrate(db); err != nil {
		t.Fatalf("migrate: %v", err)
	}

	service := NewService(db, "https://sentry.felorx.com")
	project, err := service.CreateProject(CreateProjectInput{
		Slug:         "mobile-app",
		Name:         "Mobile App",
		FelorxUserID: "user-1",
		Email:        "owner@example.com",
	})
	if err != nil {
		t.Fatalf("CreateProject returned error: %v", err)
	}

	if project.DsnPublicKey == "" {
		t.Fatal("DsnPublicKey is empty")
	}
	if project.DSN != "https://"+project.DsnPublicKey+"@sentry.felorx.com/mobile-app" {
		t.Fatalf("DSN = %s", project.DSN)
	}
}
```

- [ ] **Step 3: Run tests and verify RED**

Run:

```bash
cd apps/sentry
go test ./internal/ingest ./internal/projects
```

Expected: FAIL because DSN parser and project service do not exist.

- [ ] **Step 4: Implement DSN and project service**

Create:

```go
type ParsedDSN struct {
	PublicKey string
	ProjectID string
}

func ParseDSN(value string) (ParsedDSN, error)
```

Create project service API:

```go
type Service struct { db *gorm.DB; publicBaseURL string }
type CreateProjectInput struct { Slug, Name, Platform, FelorxAppID, FelorxAppName, FelorxUserID, Email string }
type ProjectDTO struct { ID, Slug, Name, Platform, Status, DsnPublicKey, DSN, FelorxAppID, FelorxAppName string }

func NewService(db *gorm.DB, publicBaseURL string) *Service
func (s *Service) CreateProject(input CreateProjectInput) (ProjectDTO, error)
func (s *Service) ListProjects(userID string) ([]ProjectDTO, error)
func (s *Service) RotateDSN(projectID string, userID string) (ProjectDTO, error)
```

Generate `DsnPublicKey` with cryptographically random bytes encoded as hex. Create an owner `ProjectMember` in the same transaction.

- [ ] **Step 5: Run tests**

Run:

```bash
cd apps/sentry
gofmt -w internal/ingest internal/projects
go test ./internal/ingest ./internal/projects
```

Expected: all tests pass.

- [ ] **Step 6: Commit**

```bash
cd apps/sentry
git add internal/ingest/dsn.go internal/ingest/dsn_test.go internal/projects
git commit -m "feat(projects): 增加项目和 DSN 管理"
```

---

## Task 5: Implement Event Extraction And Issue Grouping

**Files:**
- Create: `apps/sentry/internal/ingest/extractors.go`
- Create: `apps/sentry/internal/ingest/extractors_test.go`
- Create: `apps/sentry/internal/issues/grouping.go`
- Create: `apps/sentry/internal/issues/grouping_test.go`
- Create: `apps/sentry/internal/issues/service.go`
- Create: `apps/sentry/internal/issues/service_test.go`

- [ ] **Step 1: Write failing event extractor test**

Create `apps/sentry/internal/ingest/extractors_test.go`:

```go
package ingest

import "testing"

func TestExtractEventIndexesOfficialFields(t *testing.T) {
	payload := []byte(`{
	  "event_id":"fc6d8c0c43fc4630ad850ee518f1b9d0",
	  "timestamp":1304358096.0,
	  "platform":"python",
	  "level":"error",
	  "logger":"api",
	  "transaction":"/users/<username>/",
	  "release":"mobile@1.0.0",
	  "environment":"production",
	  "fingerprint":["custom","group"],
	  "contexts":{"trace":{"trace_id":"743ad8bbfdd84e99bc38b4729e2864de","span_id":"a0cfbde2bdff3adc"}},
	  "exception":{"values":[{"type":"ValueError","value":"broken"}]}
	}`)

	event, err := ExtractEvent(payload)
	if err != nil {
		t.Fatalf("ExtractEvent returned error: %v", err)
	}

	if event.EventID != "fc6d8c0c43fc4630ad850ee518f1b9d0" {
		t.Fatalf("EventID = %s", event.EventID)
	}
	if event.TraceID != "743ad8bbfdd84e99bc38b4729e2864de" {
		t.Fatalf("TraceID = %s", event.TraceID)
	}
	if event.ExceptionType != "ValueError" {
		t.Fatalf("ExceptionType = %s", event.ExceptionType)
	}
}
```

- [ ] **Step 2: Write failing grouping tests**

Create `apps/sentry/internal/issues/grouping_test.go`:

```go
package issues

import "testing"

func TestFingerprintUsesOfficialFingerprintFirst(t *testing.T) {
	event := EventForGrouping{
		Fingerprint: []string{"custom", "group"},
		Message:     "ignored",
	}

	fp := ComputeFingerprint(event)
	if fp != "fingerprint:custom|group" {
		t.Fatalf("fingerprint = %s", fp)
	}
}

func TestFingerprintFallsBackToExceptionAndTopFrame(t *testing.T) {
	event := EventForGrouping{
		ExceptionType: "ValueError",
		ExceptionValue: "broken",
		TopFrame: StackFrame{Filename: "app.py", Function: "handler", LineNo: 42},
	}

	fp := ComputeFingerprint(event)
	if fp == "" || fp == "fingerprint:" {
		t.Fatalf("fingerprint = %s", fp)
	}
}
```

- [ ] **Step 3: Run tests and verify RED**

Run:

```bash
cd apps/sentry
go test ./internal/ingest ./internal/issues
```

Expected: FAIL because extractors and grouping do not exist.

- [ ] **Step 4: Implement extractor DTOs**

In `apps/sentry/internal/ingest/extractors.go`, implement:

```go
type EventIndex struct {
	EventID       string
	Type          string
	Timestamp     time.Time
	Platform      string
	Level         string
	Logger        string
	Message       string
	Transaction   string
	ServerName    string
	Release       string
	Dist          string
	Environment   string
	Fingerprint   []string
	ExceptionType string
	TraceID       string
	SpanID        string
	UserID        string
	Raw           map[string]any
}

func ExtractEvent(payload []byte) (EventIndex, error)
```

Default `Level` to `error` and `Environment` to `production` when absent. Parse Sentry timestamps from RFC3339 strings or numeric Unix seconds.

- [ ] **Step 5: Implement grouping service**

In `apps/sentry/internal/issues/grouping.go`, implement:

```go
type StackFrame struct { Filename, Function string; LineNo int }
type EventForGrouping struct { Fingerprint []string; ExceptionType, ExceptionValue, Message, Logger, Transaction string; TopFrame StackFrame }

func ComputeFingerprint(event EventForGrouping) string
func BuildTitle(event EventForGrouping) string
```

Use `fingerprint:` prefix for official fingerprints. Use SHA-256 hex for derived fingerprints with `default:` prefix.

- [ ] **Step 6: Implement issue upsert**

In `apps/sentry/internal/issues/service.go`, implement:

```go
type Service struct { db *gorm.DB }
func NewService(db *gorm.DB) *Service
func (s *Service) UpsertIssueForEvent(projectID uuid.UUID, event storage.Event, grouping EventForGrouping) (storage.Issue, error)
```

Rules:

- Create unresolved issue for new fingerprint.
- Increment `event_count` and update `last_seen`.
- Resolved issue becomes unresolved when a new event arrives.
- Ignored issue stays ignored.

- [ ] **Step 7: Run tests**

Run:

```bash
cd apps/sentry
gofmt -w internal/ingest internal/issues
go test ./internal/ingest ./internal/issues
```

Expected: all tests pass.

- [ ] **Step 8: Commit**

```bash
cd apps/sentry
git add internal/ingest/extractors.go internal/ingest/extractors_test.go internal/issues
git commit -m "feat(issues): 增加事件抽取和 Issue 聚合"
```

---

## Task 6: Implement Log And Trace Processors

**Files:**
- Create: `apps/sentry/internal/logs/service.go`
- Create: `apps/sentry/internal/logs/service_test.go`
- Create: `apps/sentry/internal/traces/service.go`
- Create: `apps/sentry/internal/traces/service_test.go`

- [ ] **Step 1: Write failing log processor test**

Create `apps/sentry/internal/logs/service_test.go`:

```go
package logs

import "testing"

func TestParseLogItemExtractsItems(t *testing.T) {
	payload := []byte(`{
	  "version":2,
	  "items":[{
	    "timestamp":1746456149.0191,
	    "trace_id":"624f66e93a04469f9992c7e9f1485056",
	    "span_id":"b0e6f15b45c36b12",
	    "level":"info",
	    "body":"User John has logged in!",
	    "severity_number":9,
	    "attributes":{"user.id":{"value":"john","type":"string"}}
	  }]
	}`)

	items, err := ParseLogPayload(payload, 1)
	if err != nil {
		t.Fatalf("ParseLogPayload returned error: %v", err)
	}
	if items[0].Body != "User John has logged in!" {
		t.Fatalf("Body = %s", items[0].Body)
	}
	if items[0].SeverityNumber != 9 {
		t.Fatalf("SeverityNumber = %d", items[0].SeverityNumber)
	}
}
```

- [ ] **Step 2: Write failing trace processor tests**

Create `apps/sentry/internal/traces/service_test.go`:

```go
package traces

import "testing"

func TestParseTransactionPayloadExtractsRootAndSpans(t *testing.T) {
	payload := []byte(`{
	  "type":"transaction",
	  "transaction":"GET /users",
	  "start_timestamp":1304358096.242,
	  "timestamp":1304358096.955,
	  "contexts":{"trace":{"trace_id":"743ad8bbfdd84e99bc38b4729e2864de","span_id":"a0cfbde2bdff3adc","op":"http.server","status":"ok"}},
	  "spans":[{"span_id":"b01b9f6349558cd1","parent_span_id":"a0cfbde2bdff3adc","trace_id":"743ad8bbfdd84e99bc38b4729e2864de","op":"db","description":"SELECT 1","start_timestamp":1304358096.300,"timestamp":1304358096.400}]
	}`)

	trace, spans, err := ParseTransactionPayload(payload)
	if err != nil {
		t.Fatalf("ParseTransactionPayload returned error: %v", err)
	}
	if trace.TraceID != "743ad8bbfdd84e99bc38b4729e2864de" {
		t.Fatalf("TraceID = %s", trace.TraceID)
	}
	if len(spans) != 2 {
		t.Fatalf("spans = %d", len(spans))
	}
}
```

- [ ] **Step 3: Run tests and verify RED**

Run:

```bash
cd apps/sentry
go test ./internal/logs ./internal/traces
```

Expected: FAIL because processors do not exist.

- [ ] **Step 4: Implement log parser and persistence service**

Implement:

```go
type LogItem struct {
	Timestamp      time.Time
	TraceID        string
	SpanID         string
	Level          string
	Body           string
	SeverityNumber int
	Attributes     map[string]any
	Raw            map[string]any
}

func ParseLogPayload(payload []byte, expectedCount int) ([]LogItem, error)
```

If `expectedCount > 0`, return an error when `len(items)` differs.

- [ ] **Step 5: Implement transaction and span v2 parsers**

Implement:

```go
type TraceIndex struct { TraceID, Name, Operation, Status string; StartTimestamp, EndTimestamp time.Time; DurationMS float64 }
type SpanIndex struct { TraceID, SpanID, ParentSpanID, Name, Operation, Status string; IsSegment bool; StartTimestamp, EndTimestamp time.Time; DurationMS float64; Attributes, Links map[string]any; Raw map[string]any }

func ParseTransactionPayload(payload []byte) (TraceIndex, []SpanIndex, error)
func ParseSpanV2Payload(payload []byte, expectedCount int) ([]SpanIndex, error)
```

For transaction payloads, include the root transaction as a segment span.

- [ ] **Step 6: Run tests**

Run:

```bash
cd apps/sentry
gofmt -w internal/logs internal/traces
go test ./internal/logs ./internal/traces
```

Expected: all tests pass.

- [ ] **Step 7: Commit**

```bash
cd apps/sentry
git add internal/logs internal/traces
git commit -m "feat(telemetry): 解析日志和链路数据"
```

---

## Task 7: Implement Ingest Handler End-To-End

**Files:**
- Create: `apps/sentry/internal/ingest/handler.go`
- Create: `apps/sentry/internal/ingest/handler_test.go`
- Create: `apps/sentry/internal/events/service.go`
- Modify: `apps/sentry/internal/shared/response.go`

- [ ] **Step 1: Write failing ingest HTTP test**

Create `apps/sentry/internal/ingest/handler_test.go`:

```go
package ingest

import (
	"bytes"
	"net/http"
	"net/http/httptest"
	"testing"

	"github.com/felorx/felorx-sentry/internal/issues"
	"github.com/felorx/felorx-sentry/internal/projects"
	"github.com/felorx/felorx-sentry/internal/storage"
	"github.com/felorx/felorx-sentry/internal/testutil"
	"github.com/gin-gonic/gin"
)

func TestPostEnvelopeCreatesEventAndIssue(t *testing.T) {
	gin.SetMode(gin.TestMode)
	db := testutil.OpenSQLite(t)
	if err := storage.AutoMigrate(db); err != nil {
		t.Fatalf("migrate: %v", err)
	}
	project, err := projects.NewService(db, "https://sentry.felorx.com").CreateProject(projects.CreateProjectInput{
		Slug: "mobile-app", Name: "Mobile App", FelorxUserID: "user-1",
	})
	if err != nil {
		t.Fatalf("create project: %v", err)
	}

	router := gin.New()
	RegisterRoutes(router, NewHandler(db, issues.NewService(db)))

	body := []byte("{}\n{\"type\":\"event\"}\n{\"event_id\":\"fc6d8c0c43fc4630ad850ee518f1b9d0\",\"message\":\"boom\",\"level\":\"error\",\"platform\":\"python\"}\n")
	req := httptest.NewRequest(http.MethodPost, "/api/"+project.Slug+"/envelope/", bytes.NewReader(body))
	w := httptest.NewRecorder()
	router.ServeHTTP(w, req)

	if w.Code != http.StatusOK {
		t.Fatalf("status = %d body = %s", w.Code, w.Body.String())
	}

	var count int64
	db.Model(&storage.Event{}).Count(&count)
	if count != 1 {
		t.Fatalf("events = %d", count)
	}
	db.Model(&storage.Issue{}).Count(&count)
	if count != 1 {
		t.Fatalf("issues = %d", count)
	}
}
```

- [ ] **Step 2: Run test and verify RED**

Run:

```bash
cd apps/sentry
go test ./internal/ingest -run TestPostEnvelopeCreatesEventAndIssue
```

Expected: FAIL because handler routes and persistence are not implemented.

- [ ] **Step 3: Implement ingest routes**

In `apps/sentry/internal/ingest/handler.go`, expose:

```go
func RegisterRoutes(router gin.IRouter, handler *Handler)
```

Register:

```text
POST /api/:project_id/envelope/
POST /api/:project_id/api/store/envelope/
POST /api/:project_id/store/
POST /api/:project_id/api/store/
```

For envelope requests:

- Resolve active project by slug or DSN public key.
- Create `storage.Envelope`.
- Create one `storage.EnvelopeItem` per item.
- Process `event`, `transaction`, `span`, and `log`.
- Preserve unknown item.

For store requests:

- Treat request body as one event payload.
- Preserve it as an event-style envelope item with `item_type=event`.

- [ ] **Step 4: Run ingest tests**

Run:

```bash
cd apps/sentry
gofmt -w internal/ingest internal/events internal/shared
go test ./internal/ingest ./internal/events ./internal/issues ./internal/logs ./internal/traces
```

Expected: all tests pass.

- [ ] **Step 5: Commit**

```bash
cd apps/sentry
git add internal/ingest internal/events internal/shared
git commit -m "feat(ingest): 打通 Sentry 上报入库链路"
```

---

## Task 8: Add Auth, Config, And Admin API Router

**Files:**
- Modify: `apps/sentry/internal/config/config.go`
- Create: `apps/sentry/internal/auth/authenticator.go`
- Create: `apps/sentry/internal/auth/dev_authenticator.go`
- Create: `apps/sentry/internal/auth/oidc_authenticator.go`
- Create: `apps/sentry/internal/auth/middleware.go`
- Create: `apps/sentry/internal/auth/middleware_test.go`
- Create: `apps/sentry/internal/router/router.go`
- Modify: `apps/sentry/cmd/api/main.go`
- Modify: `apps/sentry/go.mod`

- [ ] **Step 1: Write failing dev auth middleware test**

Create `apps/sentry/internal/auth/middleware_test.go`:

```go
package auth

import (
	"net/http"
	"net/http/httptest"
	"testing"

	"github.com/gin-gonic/gin"
)

func TestMiddlewareAcceptsDevUserHeaderInDevMode(t *testing.T) {
	gin.SetMode(gin.TestMode)
	router := gin.New()
	router.Use(Middleware(NewDevAuthenticator()))
	router.GET("/me", func(c *gin.Context) {
		user := CurrentUser(c)
		c.JSON(http.StatusOK, gin.H{"id": user.ID, "email": user.Email})
	})

	req := httptest.NewRequest(http.MethodGet, "/me", nil)
	req.Header.Set("X-Felorx-Dev-User", "user-1")
	req.Header.Set("X-Felorx-Dev-Email", "dev@example.com")
	w := httptest.NewRecorder()
	router.ServeHTTP(w, req)

	if w.Code != http.StatusOK {
		t.Fatalf("status = %d body = %s", w.Code, w.Body.String())
	}
}
```

- [ ] **Step 2: Run test and verify RED**

Run:

```bash
cd apps/sentry
go test ./internal/auth
```

Expected: FAIL because auth middleware does not exist.

- [ ] **Step 3: Implement auth interfaces**

Create:

```go
type User struct { ID, Email, Name string }
type Authenticator interface { Authenticate(ctx context.Context, request *http.Request) (User, error) }
func Middleware(authenticator Authenticator) gin.HandlerFunc
func CurrentUser(c *gin.Context) User
```

`DevAuthenticator` reads `X-Felorx-Dev-User` and `X-Felorx-Dev-Email`. `OIDCAuthenticator` validates bearer tokens using `github.com/coreos/go-oidc/v3/oidc`.

- [ ] **Step 4: Expand config**

Modify config to include:

```go
type AuthConfig struct {
	Mode         string
	OIDCIssuer   string
	ClientID     string
	ClientSecret string
	RedirectURI  string
}

type PublicConfig struct {
	BaseURL string
	WebURL  string
}
```

Environment variables:

```text
AUTH_MODE=oidc
FELORX_OIDC_ISSUER=
FELORX_OIDC_CLIENT_ID=
FELORX_OIDC_CLIENT_SECRET=
FELORX_OIDC_REDIRECT_URI=
PUBLIC_BASE_URL=
WEB_BASE_URL=
```

- [ ] **Step 5: Create central router**

Create `internal/router/router.go`:

```go
func New(db *gorm.DB, cfg *config.Config) (*gin.Engine, error)
```

Register public ingest routes without auth and admin routes under auth middleware.

- [ ] **Step 6: Run auth tests**

Run:

```bash
cd apps/sentry
go get github.com/coreos/go-oidc/v3/oidc golang.org/x/oauth2
gofmt -w cmd internal/config internal/auth internal/router
go test ./internal/auth ./internal/router
go test ./...
```

Expected: all tests pass.

- [ ] **Step 7: Commit**

```bash
cd apps/sentry
git add go.mod go.sum cmd/api/main.go internal/config internal/auth internal/router
git commit -m "feat(auth): 接入 Felorx OIDC 认证边界"
```

---

## Task 9: Add Admin API Handlers For Projects, Issues, Events, Logs, And Traces

**Files:**
- Create: `apps/sentry/internal/projects/handler.go`
- Create: `apps/sentry/internal/events/handler.go`
- Create: `apps/sentry/internal/issues/handler.go`
- Create: `apps/sentry/internal/logs/handler.go`
- Create: `apps/sentry/internal/traces/handler.go`
- Create: handler tests in each package
- Modify: `apps/sentry/internal/router/router.go`

- [ ] **Step 1: Write failing project API test**

Create `apps/sentry/internal/projects/handler_test.go`:

```go
package projects

import (
	"net/http"
	"net/http/httptest"
	"strings"
	"testing"

	"github.com/felorx/felorx-sentry/internal/auth"
	"github.com/felorx/felorx-sentry/internal/storage"
	"github.com/felorx/felorx-sentry/internal/testutil"
	"github.com/gin-gonic/gin"
)

func TestCreateProjectEndpoint(t *testing.T) {
	gin.SetMode(gin.TestMode)
	db := testutil.OpenSQLite(t)
	_ = storage.AutoMigrate(db)
	router := gin.New()
	router.Use(auth.Middleware(auth.NewDevAuthenticator()))
	RegisterRoutes(router.Group("/api/projects"), NewHandler(NewService(db, "https://sentry.felorx.com")))

	req := httptest.NewRequest(http.MethodPost, "/api/projects", strings.NewReader(`{"slug":"api","name":"API"}`))
	req.Header.Set("Content-Type", "application/json")
	req.Header.Set("X-Felorx-Dev-User", "user-1")
	w := httptest.NewRecorder()
	router.ServeHTTP(w, req)

	if w.Code != http.StatusOK {
		t.Fatalf("status = %d body = %s", w.Code, w.Body.String())
	}
}
```

- [ ] **Step 2: Run handler tests and verify RED**

Run:

```bash
cd apps/sentry
go test ./internal/projects -run TestCreateProjectEndpoint
```

Expected: FAIL because handlers do not exist.

- [ ] **Step 3: Implement admin handlers**

Expose routes:

```text
GET    /api/projects
POST   /api/projects
GET    /api/projects/:project_id
PATCH  /api/projects/:project_id
POST   /api/projects/:project_id/rotate-dsn
GET    /api/projects/:project_id/overview
GET    /api/projects/:project_id/issues
GET    /api/projects/:project_id/issues/:issue_id
PATCH  /api/projects/:project_id/issues/:issue_id
GET    /api/projects/:project_id/events
GET    /api/projects/:project_id/events/:event_id
GET    /api/projects/:project_id/logs
GET    /api/projects/:project_id/traces
GET    /api/projects/:project_id/traces/:trace_id
```

Every handler must resolve project access from `project_members`. Use `viewer` for reads, `member` for issue status changes, and `admin` or `owner` for project settings and DSN rotation.

- [ ] **Step 4: Implement pagination and filters**

Create shared query helpers for:

```text
page
page_size
query
environment
release
level
status
start
end
```

Use `page_size` max 100.

- [ ] **Step 5: Run admin API tests**

Run:

```bash
cd apps/sentry
gofmt -w internal/projects internal/events internal/issues internal/logs internal/traces internal/router
go test ./internal/projects ./internal/events ./internal/issues ./internal/logs ./internal/traces ./internal/router
```

Expected: all tests pass.

- [ ] **Step 6: Commit**

```bash
cd apps/sentry
git add internal/projects internal/events internal/issues internal/logs internal/traces internal/router
git commit -m "feat(api): 增加 Sentry 后台管理接口"
```

---

## Task 10: Add Next.js Auth And API Client

**Files:**
- Modify: `apps/sentry/web/package.json`
- Create: `apps/sentry/web/vitest.config.ts`
- Create: `apps/sentry/web/test/setup.ts`
- Create: `apps/sentry/web/lib/api/types.ts`
- Create: `apps/sentry/web/lib/api/client.ts`
- Create: `apps/sentry/web/lib/api/__tests__/client.test.ts`
- Create: `apps/sentry/web/lib/auth/options.ts`
- Create: `apps/sentry/web/app/api/auth/[...nextauth]/route.ts`

- [ ] **Step 1: Add test dependencies**

Run:

```bash
cd apps/sentry/web
npm install next-auth
npm install -D vitest @vitejs/plugin-react jsdom @testing-library/react @testing-library/jest-dom
```

Modify `package.json` scripts:

```json
{
  "scripts": {
    "dev": "next dev --turbopack",
    "build": "next build",
    "start": "next start",
    "lint": "next lint",
    "test": "vitest run",
    "test:watch": "vitest"
  }
}
```

- [ ] **Step 2: Write failing API client tests**

Create `apps/sentry/web/lib/api/__tests__/client.test.ts`:

```ts
import { describe, expect, it, vi } from "vitest";
import { SentryApiClient } from "../client";

describe("SentryApiClient", () => {
  it("builds project issue query parameters", async () => {
    const fetchMock = vi.fn().mockResolvedValue({
      ok: true,
      json: async () => ({ data: [], total: 0 }),
    });
    const client = new SentryApiClient("https://api.example.com", async () => "token", fetchMock);

    await client.listIssues("project-1", { status: "unresolved", level: "error", page: 2 });

    expect(fetchMock).toHaveBeenCalledWith(
      "https://api.example.com/api/projects/project-1/issues?status=unresolved&level=error&page=2",
      expect.objectContaining({
        headers: expect.objectContaining({ Authorization: "Bearer token" }),
      }),
    );
  });
});
```

- [ ] **Step 3: Run test and verify RED**

Run:

```bash
cd apps/sentry/web
npm test -- lib/api/__tests__/client.test.ts
```

Expected: FAIL because `SentryApiClient` does not exist.

- [ ] **Step 4: Implement API types and client**

Create `types.ts` with DTOs for `Project`, `Issue`, `Event`, `LogEntry`, `TraceSummary`, `TraceDetail`, `PaginatedResult<T>`.

Create `client.ts`:

```ts
export class SentryApiClient {
  constructor(
    private readonly baseUrl: string,
    private readonly getToken: () => Promise<string | null>,
    private readonly fetcher: typeof fetch = fetch,
  ) {}

  async listIssues(projectId: string, filters: Record<string, string | number | undefined>) {
    return this.get(`/api/projects/${projectId}/issues`, filters);
  }

  private async get<T>(path: string, query?: Record<string, string | number | undefined>): Promise<T> {
    const token = await this.getToken();
    const url = new URL(path, this.baseUrl);
    Object.entries(query ?? {}).forEach(([key, value]) => {
      if (value !== undefined && value !== "") url.searchParams.set(key, String(value));
    });
    const response = await this.fetcher(url.toString(), {
      headers: token ? { Authorization: `Bearer ${token}` } : {},
    });
    if (!response.ok) throw new Error(`API request failed: ${response.status}`);
    return response.json() as Promise<T>;
  }
}
```

- [ ] **Step 5: Implement Auth.js OIDC options**

Create `lib/auth/options.ts` and `app/api/auth/[...nextauth]/route.ts` using `next-auth` OIDC provider configured by:

```text
FELORX_OIDC_ISSUER
FELORX_OIDC_CLIENT_ID
FELORX_OIDC_CLIENT_SECRET
NEXTAUTH_SECRET
NEXT_PUBLIC_API_URL
```

JWT callback must retain `access_token` for API calls.

- [ ] **Step 6: Run tests and build**

Run:

```bash
cd apps/sentry/web
npm test
npm run build
```

Expected: tests and build pass.

- [ ] **Step 7: Commit**

```bash
cd apps/sentry
git add web/package.json web/package-lock.json web/vitest.config.ts web/test web/lib web/app/api/auth
git commit -m "feat(web): 接入 Felorx 登录和后台 API 客户端"
```

---

## Task 11: Build Admin UI Pages

**Files:**
- Modify: `apps/sentry/web/app/layout.tsx`
- Replace: `apps/sentry/web/app/page.tsx`
- Replace: `apps/sentry/web/app/events/[id]/page.tsx`
- Create: all project, issue, event, log, trace pages listed in File Structure
- Create: `apps/sentry/web/components/AppShell.tsx`
- Create: `apps/sentry/web/components/JsonPanel.tsx`
- Create: `apps/sentry/web/components/StatusBadge.tsx`
- Create: `apps/sentry/web/components/TraceTree.tsx`
- Create: component tests

- [ ] **Step 1: Write failing component tests**

Create `apps/sentry/web/components/__tests__/StatusBadge.test.tsx`:

```tsx
import { render, screen } from "@testing-library/react";
import { describe, expect, it } from "vitest";
import { StatusBadge } from "../StatusBadge";

describe("StatusBadge", () => {
  it("renders unresolved issue status", () => {
    render(<StatusBadge value="unresolved" />);
    expect(screen.getByText("unresolved")).toBeInTheDocument();
  });
});
```

Create `apps/sentry/web/components/__tests__/TraceTree.test.tsx`:

```tsx
import { render, screen } from "@testing-library/react";
import { describe, expect, it } from "vitest";
import { TraceTree } from "../TraceTree";

describe("TraceTree", () => {
  it("renders parent and child spans", () => {
    render(<TraceTree spans={[
      { span_id: "root", name: "GET /users", trace_id: "trace", is_segment: true },
      { span_id: "child", parent_span_id: "root", name: "db.query", trace_id: "trace", is_segment: false },
    ]} />);
    expect(screen.getByText("GET /users")).toBeInTheDocument();
    expect(screen.getByText("db.query")).toBeInTheDocument();
  });
});
```

- [ ] **Step 2: Run tests and verify RED**

Run:

```bash
cd apps/sentry/web
npm test -- components/__tests__
```

Expected: FAIL because components do not exist.

- [ ] **Step 3: Implement shared components**

Implement:

- `AppShell`: top nav, project nav, user menu, sign out.
- `JsonPanel`: collapsible JSON viewer with `JSON.stringify(value, null, 2)`.
- `StatusBadge`: color-coded `active/disabled/unresolved/resolved/ignored/fatal/error/warning/info/debug`.
- `TraceTree`: deterministic tree built from `span_id` and `parent_span_id`.

Avoid decorative landing-page composition; first screen should be the project list or selected project dashboard.

- [ ] **Step 4: Replace page routes**

Implement pages:

```text
/login
/
/projects
/projects/[projectId]
/projects/[projectId]/settings
/projects/[projectId]/issues
/projects/[projectId]/issues/[issueId]
/projects/[projectId]/events/[eventId]
/projects/[projectId]/logs
/projects/[projectId]/traces
/projects/[projectId]/traces/[traceId]
```

Each page must handle loading, empty, and error states. Lists must expose filters matching the admin API.

- [ ] **Step 5: Run UI tests and build**

Run:

```bash
cd apps/sentry/web
npm test
npm run lint
npm run build
```

Expected: tests, lint, and build pass.

- [ ] **Step 6: Commit**

```bash
cd apps/sentry
git add web/app web/components web/lib
git commit -m "feat(web): 重构 Sentry 后台管理界面"
```

---

## Task 12: End-To-End Verification, Docker, And Docs

**Files:**
- Modify: `apps/sentry/README.md`
- Modify: `apps/sentry/docker-compose.yml`
- Modify: `apps/sentry/Dockerfile`
- Modify: `apps/sentry/web/Dockerfile`
- Create: `apps/sentry/.env.example`
- Create: `apps/sentry/internal/ingest/e2e_test.go`

- [ ] **Step 1: Write failing end-to-end ingest test**

Create `apps/sentry/internal/ingest/e2e_test.go`:

```go
package ingest

import (
	"bytes"
	"net/http"
	"net/http/httptest"
	"testing"

	"github.com/felorx/felorx-sentry/internal/issues"
	"github.com/felorx/felorx-sentry/internal/projects"
	"github.com/felorx/felorx-sentry/internal/storage"
	"github.com/felorx/felorx-sentry/internal/testutil"
	"github.com/gin-gonic/gin"
)

func TestMVPIngestStoresOfficialPayloadIssueLogAndTrace(t *testing.T) {
	gin.SetMode(gin.TestMode)
	db := testutil.OpenSQLite(t)
	if err := storage.AutoMigrate(db); err != nil {
		t.Fatalf("migrate: %v", err)
	}

	project, err := projects.NewService(db, "https://sentry.felorx.com").CreateProject(projects.CreateProjectInput{
		Slug: "mobile-app",
		Name: "Mobile App",
		FelorxUserID: "user-1",
		Email: "owner@example.com",
	})
	if err != nil {
		t.Fatalf("create project: %v", err)
	}

	router := gin.New()
	RegisterRoutes(router, NewHandler(db, issues.NewService(db)))

	postEnvelope := func(body string) {
		t.Helper()
		req := httptest.NewRequest(http.MethodPost, "/api/"+project.Slug+"/envelope/", bytes.NewReader([]byte(body)))
		w := httptest.NewRecorder()
		router.ServeHTTP(w, req)
		if w.Code != http.StatusOK {
			t.Fatalf("status = %d body = %s", w.Code, w.Body.String())
		}
	}

	postEnvelope("{}\n{\"type\":\"event\"}\n{\"event_id\":\"fc6d8c0c43fc4630ad850ee518f1b9d0\",\"timestamp\":1304358096.0,\"platform\":\"python\",\"level\":\"error\",\"message\":\"boom\",\"fingerprint\":[\"boom\"],\"contexts\":{\"trace\":{\"trace_id\":\"743ad8bbfdd84e99bc38b4729e2864de\",\"span_id\":\"a0cfbde2bdff3adc\"}}}\n")
	postEnvelope("{}\n{\"type\":\"log\",\"item_count\":1,\"content_type\":\"application/vnd.sentry.items.log+json\"}\n{\"version\":2,\"items\":[{\"timestamp\":1746456149.0191,\"trace_id\":\"743ad8bbfdd84e99bc38b4729e2864de\",\"span_id\":\"a0cfbde2bdff3adc\",\"level\":\"info\",\"body\":\"request started\",\"severity_number\":9}]}\n")
	postEnvelope("{}\n{\"type\":\"transaction\"}\n{\"type\":\"transaction\",\"transaction\":\"GET /users\",\"start_timestamp\":1304358096.242,\"timestamp\":1304358096.955,\"contexts\":{\"trace\":{\"trace_id\":\"743ad8bbfdd84e99bc38b4729e2864de\",\"span_id\":\"a0cfbde2bdff3adc\",\"op\":\"http.server\",\"status\":\"ok\"}},\"spans\":[{\"span_id\":\"b01b9f6349558cd1\",\"parent_span_id\":\"a0cfbde2bdff3adc\",\"trace_id\":\"743ad8bbfdd84e99bc38b4729e2864de\",\"op\":\"db\",\"description\":\"SELECT 1\",\"start_timestamp\":1304358096.300,\"timestamp\":1304358096.400}]}\n")

	assertCount := func(model any, expected int64) {
		t.Helper()
		var count int64
		db.Model(model).Count(&count)
		if count != expected {
			t.Fatalf("%T count = %d, want %d", model, count, expected)
		}
	}

	assertCount(&storage.Event{}, 2)
	assertCount(&storage.Issue{}, 1)
	assertCount(&storage.Log{}, 1)
	assertCount(&storage.Trace{}, 1)
	assertCount(&storage.Span{}, 2)
}
```

The test creates a project, posts event/log/transaction envelopes with the same `trace_id`, and asserts event, issue, log, trace, and span rows exist.

- [ ] **Step 2: Update Docker and environment files**

`.env.example` must include:

```text
SERVER_ADDRESS=:8012
PUBLIC_BASE_URL=http://localhost:8012
WEB_BASE_URL=http://localhost:3000
DB_HOST=postgres
DB_PORT=5432
DB_USER=felorx_sentry
DB_PASSWORD=felorx_sentry
DB_NAME=felorx_sentry
DB_SSL_MODE=disable
AUTH_MODE=dev
FELORX_OIDC_ISSUER=
FELORX_OIDC_CLIENT_ID=
FELORX_OIDC_CLIENT_SECRET=
FELORX_OIDC_REDIRECT_URI=http://localhost:3000/api/auth/callback/felorx
NEXTAUTH_SECRET=dev-secret-change-me
NEXT_PUBLIC_API_URL=http://localhost:8012
```

`docker-compose.yml` must run `postgres`, `api`, and `web` with the same variable names.

- [ ] **Step 3: Update README**

Document:

- Submodule location.
- Local start with docker-compose.
- Felorx OIDC configuration.
- Dev auth mode.
- DSN examples.
- Supported Sentry item types.
- Verification commands.

- [ ] **Step 4: Run full verification**

Run:

```bash
cd apps/sentry
go test ./...
docker compose config
cd web
npm test
npm run lint
npm run build
```

Expected: all commands pass.

- [ ] **Step 5: Smoke test local stack**

Run:

```bash
cd apps/sentry
docker compose up -d --build
curl -f http://localhost:8012/health
curl -f http://localhost:3000
docker compose down
```

Expected: API health and web root return successful HTTP responses.

- [ ] **Step 6: Commit submodule and parent pointer**

Inside submodule:

```bash
cd apps/sentry
git add README.md docker-compose.yml Dockerfile web/Dockerfile .env.example internal/ingest/e2e_test.go
git commit -m "docs(sentry): 完善本地部署和验收说明"
```

In root:

```bash
git add apps/sentry
git commit -m "feat(sentry): 更新 Felorx Sentry MVP 子模块"
```

---

## Final Verification Checklist

- [ ] `git submodule status apps/sentry` shows `apps/sentry` mapped to `git@github.com:felorx/felorx-sentry.git`.
- [ ] `cd apps/sentry && go test ./...` passes.
- [ ] `cd apps/sentry/web && npm test` passes.
- [ ] `cd apps/sentry/web && npm run lint` passes.
- [ ] `cd apps/sentry/web && npm run build` passes.
- [ ] Event ingest preserves official event raw payload.
- [ ] Unknown envelope item is stored in `envelope_items`.
- [ ] Official `fingerprint` takes precedence over derived grouping.
- [ ] `log` item `items[]` entries are searchable by `level` and `trace_id`.
- [ ] Transaction payload and span v2 item both appear in trace detail.
- [ ] Admin API blocks unauthenticated requests.
- [ ] Dev auth mode is available only when `AUTH_MODE=dev`.
- [ ] Docker Compose stack starts API, web, and PostgreSQL.
