# PATCH AllowNonScimKeys Support — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make `AllowNonScimKeys` work for PATCH requests in the SCIM library, then remove the workaround code in scalekit that was compensating for this gap.

**Architecture:** Two phases across two repos:
- **Phase 1** (`~/Sandbox/scim`, branch `scim-patch-custom-attr`): Thread `AllowNonScimKeys` through the PATCH validation pipeline so non-standard attributes pass through as-is.
- **Phase 2** (`~/Sandbox/scalekit3`, branch `scim-handler-patch` off `scim-handler`): Update the SCIM library dependency, remove the workaround functions that rewrote unknown attributes into `customAttributes`, and simplify the Patch handler.

**Tech Stack:** Go, SCIM v2.0 (RFC 7644)

---

# Phase 1: SCIM Library Fix

**Repo:** `~/Sandbox/scim`
**Branch:** `scim-patch-custom-attr` (off `master`)

## File Map

| File | Action | Responsibility |
|------|--------|----------------|
| `internal/patch/patch.go` | Modify | Add `allowNonScimKeys` field to `OperationValidator`; add `ValidatorConfig` + `NewValidatorWithConfig`; modify `getRefAttribute` and `validateEmptyPath` |
| `internal/patch/update.go` | Modify | Handle nil `refAttr` (non-SCIM key) — return value as-is before accessing `SubAttributes()` |
| `internal/patch/remove.go` | Modify | Same nil `refAttr` handling as update.go |
| `internal/patch/patch_test.go` | Modify | Add unit tests for non-SCIM keys with the new flag |
| `resource_type.go` | Modify | Pass `AllowNonScimKeys` to `patch.NewValidatorWithConfig` |
| `handlers_test.go` | Modify | Add end-to-end PATCH tests with custom attributes |
| `schema/schema.go` | Modify | Add variadic `allowNonScimKeys` param to `ValidatePatchOperation` |
| `schema/schema_test.go` | Modify | Add tests for `ValidatePatchOperation` with the new flag |

---

### Task 1: Create the branch

- [ ] **Step 1: Create and switch to branch**

```bash
cd ~/Sandbox/scim
git checkout -b scim-patch-custom-attr
```

- [ ] **Step 2: Verify clean state**

```bash
git status
```

Expected: clean working tree on `scim-patch-custom-attr`

---

### Task 2: Add `allowNonScimKeys` to `OperationValidator` and update `NewValidator` signature

**Files:**
- Modify: `internal/patch/patch.go:28-35` (struct), `internal/patch/patch.go:38-100` (NewValidator)
- Modify: `internal/patch/patch_test.go`

- [ ] **Step 1: Write the failing test in `internal/patch/patch_test.go`**

Add a test that creates a validator with `allowNonScimKeys=true` and a non-schema attribute path.

```go
func TestNewValidator_AllowNonScimKeys(t *testing.T) {
	// A path that does NOT exist in patchSchema.
	op, _ := json.Marshal(map[string]interface{}{
		"op":    "replace",
		"path":  "custom_attribute",
		"value": "custom_value",
	})

	// Without AllowNonScimKeys, should fail.
	t.Run("Reject unknown path by default", func(t *testing.T) {
		_, err := NewValidator(op, patchSchema)
		if err == nil {
			t.Error("expected error for unknown attribute, got none")
		}
	})

	// With AllowNonScimKeys, should succeed.
	t.Run("Accept unknown path when AllowNonScimKeys is true", func(t *testing.T) {
		validator, err := NewValidatorWithConfig(op, patchSchema, ValidatorConfig{AllowNonScimKeys: true})
		if err != nil {
			t.Fatalf("unexpected error: %v", err)
		}
		if validator.Path == nil {
			t.Fatal("expected path to be set")
		}
	})
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `go test ./internal/patch/ -run TestNewValidator_AllowNonScimKeys -v`
Expected: compilation error — `NewValidatorWithConfig` does not exist

- [ ] **Step 3: Implement the changes in `internal/patch/patch.go`**

Add `ValidatorConfig` struct and `NewValidatorWithConfig` function. Add `allowNonScimKeys` field to `OperationValidator`. `NewValidator` delegates to `NewValidatorWithConfig` for backward compatibility.

Key behavior: `filter.NewPathValidator` parses the path at construction time (via `filter.ParsePath`), then `Validate()` checks schema constraints. So `validator.Path()` returns a valid parsed path even when `Validate()` fails — the parsing succeeds for simple attribute names; only schema validation fails.

```go
// ValidatorConfig holds optional configuration for the OperationValidator.
type ValidatorConfig struct {
	// AllowNonScimKeys allows attributes not defined in the schema to pass through without validation.
	AllowNonScimKeys bool
}

// OperationValidator represents a validator to validate PATCH requests.
type OperationValidator struct {
	Op    Op
	Path  *filter.Path
	value interface{}

	schema           schema.Schema
	schemas          map[string]schema.Schema
	allowNonScimKeys bool
}

func NewValidator(patchReq []byte, s schema.Schema, extensions ...schema.Schema) (OperationValidator, error) {
	return NewValidatorWithConfig(patchReq, s, ValidatorConfig{}, extensions...)
}

func NewValidatorWithConfig(patchReq []byte, s schema.Schema, config ValidatorConfig, extensions ...schema.Schema) (OperationValidator, error) {
	var operation struct {
		Op    string
		Path  string
		Value interface{}
	}

	d := json.NewDecoder(bytes.NewReader(patchReq))
	d.UseNumber()
	if err := d.Decode(&operation); err != nil {
		return OperationValidator{}, err
	}

	operation.Op = strings.ToLower(operation.Op)

	switch v := operation.Value.(type) {
	case map[string]interface{}:
		var key string
		var found bool
		for k := range v {
			if strings.ToLower(k) == "id" {
				if found {
					return OperationValidator{}, fmt.Errorf("duplicate attributes: %s and %s", k, key)
				}
				found = true
				key = k
			}
		}
		delete(v, key)
	}

	var path *filter.Path
	if operation.Path != "" {
		validator, err := f.NewPathValidator(operation.Path, s, extensions...)
		if err != nil {
			return OperationValidator{}, err
		}
		if err := validator.Validate(); err != nil {
			if !config.AllowNonScimKeys {
				return OperationValidator{}, err
			}
		}
		p := validator.Path()
		path = &p
	}

	schemas := map[string]schema.Schema{
		s.ID: s,
	}
	for _, e := range extensions {
		schemas[e.ID] = e
	}
	return OperationValidator{
		Op:    Op(operation.Op),
		Path:  path,
		value: operation.Value,

		schema:           s,
		schemas:          schemas,
		allowNonScimKeys: config.AllowNonScimKeys,
	}, nil
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `go test ./internal/patch/ -run TestNewValidator_AllowNonScimKeys -v`
Expected: PASS

- [ ] **Step 5: Run all existing tests to verify no regressions**

Run: `go test ./internal/patch/ -v`
Expected: all PASS

- [ ] **Step 6: Commit**

```bash
git add internal/patch/patch.go internal/patch/patch_test.go
git commit -m "feat: add AllowNonScimKeys support to patch OperationValidator

Add ValidatorConfig and NewValidatorWithConfig to allow non-SCIM attribute
paths to pass through without schema validation errors."
```

---

### Task 3: Make `getRefAttribute` and validation methods handle unknown attributes

**Files:**
- Modify: `internal/patch/patch.go:122-204` (getRefAttribute, validateEmptyPath)
- Modify: `internal/patch/update.go` (validateUpdate)
- Modify: `internal/patch/remove.go` (validateRemove)
- Modify: `internal/patch/patch_test.go`

- [ ] **Step 1: Write failing tests in `internal/patch/patch_test.go`**

Test that `Validate()` succeeds for add/replace/remove on non-schema attributes when `AllowNonScimKeys` is true:

```go
func TestOperationValidator_Validate_AllowNonScimKeys(t *testing.T) {
	t.Run("Add with explicit path", func(t *testing.T) {
		op, _ := json.Marshal(map[string]interface{}{
			"op":    "add",
			"path":  "custom_attribute",
			"value": "custom_value",
		})
		validator, err := NewValidatorWithConfig(op, patchSchema, ValidatorConfig{AllowNonScimKeys: true})
		if err != nil {
			t.Fatal(err)
		}
		value, err := validator.Validate()
		if err != nil {
			t.Fatalf("unexpected error: %v", err)
		}
		if value != "custom_value" {
			t.Errorf("expected custom_value, got %v", value)
		}
	})

	t.Run("Replace with explicit path", func(t *testing.T) {
		op, _ := json.Marshal(map[string]interface{}{
			"op":    "replace",
			"path":  "custom_attribute",
			"value": "new_value",
		})
		validator, err := NewValidatorWithConfig(op, patchSchema, ValidatorConfig{AllowNonScimKeys: true})
		if err != nil {
			t.Fatal(err)
		}
		value, err := validator.Validate()
		if err != nil {
			t.Fatalf("unexpected error: %v", err)
		}
		if value != "new_value" {
			t.Errorf("expected new_value, got %v", value)
		}
	})

	t.Run("Remove with explicit path", func(t *testing.T) {
		op, _ := json.Marshal(map[string]interface{}{
			"op":   "remove",
			"path": "custom_attribute",
		})
		validator, err := NewValidatorWithConfig(op, patchSchema, ValidatorConfig{AllowNonScimKeys: true})
		if err != nil {
			t.Fatal(err)
		}
		value, err := validator.Validate()
		if err != nil {
			t.Fatalf("unexpected error: %v", err)
		}
		if value != nil {
			t.Errorf("expected nil, got %v", value)
		}
	})

	t.Run("Add with empty path containing non-SCIM keys", func(t *testing.T) {
		op, _ := json.Marshal(map[string]interface{}{
			"op": "add",
			"value": map[string]interface{}{
				"attr1":            "known_value",
				"custom_attribute": "custom_value",
			},
		})
		validator, err := NewValidatorWithConfig(op, patchSchema, ValidatorConfig{AllowNonScimKeys: true})
		if err != nil {
			t.Fatal(err)
		}
		value, err := validator.Validate()
		if err != nil {
			t.Fatalf("unexpected error: %v", err)
		}
		m, ok := value.(map[string]interface{})
		if !ok {
			t.Fatalf("expected map, got %T", value)
		}
		if m["attr1"] != "known_value" {
			t.Errorf("expected known_value, got %v", m["attr1"])
		}
		if m["custom_attribute"] != "custom_value" {
			t.Errorf("expected custom_value, got %v", m["custom_attribute"])
		}
	})

	t.Run("Add with boolean custom attribute", func(t *testing.T) {
		op, _ := json.Marshal(map[string]interface{}{
			"op":    "add",
			"path":  "custom_flag",
			"value": true,
		})
		validator, err := NewValidatorWithConfig(op, patchSchema, ValidatorConfig{AllowNonScimKeys: true})
		if err != nil {
			t.Fatal(err)
		}
		value, err := validator.Validate()
		if err != nil {
			t.Fatalf("unexpected error: %v", err)
		}
		if value != true {
			t.Errorf("expected true, got %v", value)
		}
	})

	t.Run("Add with object custom attribute", func(t *testing.T) {
		op, _ := json.Marshal(map[string]interface{}{
			"op":   "add",
			"path": "custom_object",
			"value": map[string]interface{}{
				"key": "value",
			},
		})
		validator, err := NewValidatorWithConfig(op, patchSchema, ValidatorConfig{AllowNonScimKeys: true})
		if err != nil {
			t.Fatal(err)
		}
		value, err := validator.Validate()
		if err != nil {
			t.Fatalf("unexpected error: %v", err)
		}
		m, ok := value.(map[string]interface{})
		if !ok {
			t.Fatalf("expected map, got %T", value)
		}
		if m["key"] != "value" {
			t.Errorf("expected value, got %v", m["key"])
		}
	})

	t.Run("Known attributes still validated when AllowNonScimKeys is true", func(t *testing.T) {
		// attr2 is an integer attribute — a non-integer value should still be rejected.
		op, _ := json.Marshal(map[string]interface{}{
			"op":    "add",
			"path":  "attr2",
			"value": "not_an_integer",
		})
		validator, err := NewValidatorWithConfig(op, patchSchema, ValidatorConfig{AllowNonScimKeys: true})
		if err != nil {
			t.Fatal(err)
		}
		_, err = validator.Validate()
		if err == nil {
			t.Error("expected validation error for invalid integer value, got none")
		}
	})
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `go test ./internal/patch/ -run TestOperationValidator_Validate_AllowNonScimKeys -v`
Expected: FAIL — `getRefAttribute` returns error for unknown attributes

- [ ] **Step 3: Modify `getRefAttribute` in `internal/patch/patch.go`**

When the attribute is not found and `allowNonScimKeys` is true, return `nil, nil` instead of an error. A nil `refAttr` signals "pass through without validation":

```go
func (v OperationValidator) getRefAttribute(attrPath filter.AttributePath) (*schema.CoreAttribute, error) {
	var refSchema = v.schema
	if uri := attrPath.URI(); uri != "" {
		var ok bool
		if refSchema, ok = v.schemas[uri]; !ok {
			if v.allowNonScimKeys {
				return nil, nil
			}
			return nil, fmt.Errorf("invalid uri prefix: %s", uri)
		}
	}

	var (
		refAttr  *schema.CoreAttribute
		attrName = attrPath.AttributeName
	)
	for _, attr := range refSchema.Attributes {
		if strings.EqualFold(attr.Name(), attrName) {
			refAttr = &attr
			break
		}
	}
	if refAttr == nil {
		if v.allowNonScimKeys {
			return nil, nil
		}
		return nil, fmt.Errorf("could not find attribute %s", v.Path)
	}
	if subAttrName := attrPath.SubAttributeName(); subAttrName != "" {
		refSubAttr, err := v.getRefSubAttribute(refAttr, subAttrName)
		if err != nil {
			if v.allowNonScimKeys {
				return nil, nil
			}
			return nil, err
		}
		refAttr = refSubAttr
	}
	return refAttr, nil
}
```

- [ ] **Step 4: Modify `validateUpdate` in `internal/patch/update.go`**

**Critical:** The `if refAttr == nil` check MUST come BEFORE any code that accesses `refAttr.SubAttributes()` (the `ValueExpression` and `SubAttributeName` blocks), otherwise it will panic. This is a full replacement of the function — the only change is the 4-line nil check after `getRefAttribute`:

```go
func (v OperationValidator) validateUpdate() (interface{}, error) {
	// The operation must contain a "value" member whose content specifies the value to be added/replaces.
	if v.value == nil {
		return nil, fmt.Errorf("an add operation must contain a value member")
	}

	// If "path" is omitted, the target location is assumed to be the resource itself.
	if v.Path == nil {
		return v.validateEmptyPath()
	}

	refAttr, err := v.getRefAttribute(v.Path.AttributePath)
	if err != nil {
		return nil, err
	}
	// Non-SCIM key: pass through without validation.
	if refAttr == nil {
		return v.value, nil
	}

	if v.Path.ValueExpression != nil {
		if err := f.NewFilterValidator(v.Path.ValueExpression, schema.Schema{
			Attributes: refAttr.SubAttributes(),
		}).Validate(); err != nil {
			return nil, err
		}
	}
	if subAttrName := v.Path.SubAttributeName(); subAttrName != "" {
		refSubAttr, err := v.getRefSubAttribute(refAttr, subAttrName)
		if err != nil {
			return nil, err
		}
		refAttr = refSubAttr
	}

	if !refAttr.MultiValued() {
		attr, scimErr := refAttr.ValidateSingular(v.value)
		if scimErr != nil {
			return nil, scimErr
		}
		return attr, nil
	}

	if list, ok := v.value.([]interface{}); ok {
		var attrs []interface{}
		for _, value := range list {
			attr, scimErr := refAttr.ValidateSingular(value)
			if scimErr != nil {
				return nil, scimErr
			}
			attrs = append(attrs, attr)
		}
		return attrs, nil
	}

	attr, scimErr := refAttr.ValidateSingular(v.value)
	if scimErr != nil {
		return nil, scimErr
	}
	return []interface{}{attr}, nil
}
```

- [ ] **Step 5: Modify `validateRemove` in `internal/patch/remove.go`**

Same pattern — the nil check MUST come before `ValueExpression`/`SubAttributeName` blocks. Full replacement, only change is the 4-line nil check:

```go
func (v OperationValidator) validateRemove() (interface{}, error) {
	// If "path" is unspecified, the operation fails with HTTP status code 400 and a "scimType" error code of "noTarget".
	if v.Path == nil {
		return nil, &errors.ScimError{
			ScimType: errors.ScimTypeNoTarget,
			Status:   http.StatusBadRequest,
		}
	}

	refAttr, err := v.getRefAttribute(v.Path.AttributePath)
	if err != nil {
		return nil, err
	}
	// Non-SCIM key: pass through without validation.
	if refAttr == nil {
		return v.value, nil
	}

	if v.Path.ValueExpression != nil {
		if err := filter.NewFilterValidator(v.Path.ValueExpression, schema.Schema{
			Attributes: filter.MultiValuedFilterAttributes(*refAttr),
		}).Validate(); err != nil {
			return nil, err
		}
	}
	if subAttrName := v.Path.SubAttributeName(); subAttrName != "" {
		refSubAttr, err := v.getRefSubAttribute(refAttr, subAttrName)
		if err != nil {
			return nil, err
		}
		refAttr = refSubAttr
	}
	if v.value == nil {
		return nil, nil
	}
	if !refAttr.MultiValued() {
		attr, scimErr := refAttr.ValidateSingular(v.value)
		if scimErr != nil {
			return nil, scimErr
		}
		return attr, nil
	}

	if list, ok := v.value.([]interface{}); ok {
		var attrs []interface{}
		for _, value := range list {
			attr, scimErr := refAttr.ValidateSingular(value)
			if scimErr != nil {
				return nil, scimErr
			}
			attrs = append(attrs, attr)
		}
		return attrs, nil
	}

	attr, scimErr := refAttr.ValidateSingular(v.value)
	if scimErr != nil {
		return nil, scimErr
	}
	return []interface{}{attr}, nil
}
```

- [ ] **Step 6: Modify `validateEmptyPath` in `internal/patch/patch.go`**

When an attribute key fails path parsing or validation, and `allowNonScimKeys` is true, pass the value through as-is. Also propagate `allowNonScimKeys` to the child validator and fix variable shadowing (`v` → `validated`):

```go
func (v OperationValidator) validateEmptyPath() (interface{}, error) {
	attributes, ok := v.value.(map[string]interface{})
	if !ok {
		return nil, fmt.Errorf("the given value should be a complex attribute if path is empty")
	}

	rootValue := map[string]interface{}{}
	for p, value := range attributes {
		path, err := filter.ParsePath([]byte(p))
		if err != nil {
			if v.allowNonScimKeys {
				rootValue[p] = value
				continue
			}
			return nil, fmt.Errorf("invalid attribute path: %s", p)
		}
		validator := OperationValidator{
			Op:               v.Op,
			Path:             &path,
			value:            value,
			schema:           v.schema,
			schemas:          v.schemas,
			allowNonScimKeys: v.allowNonScimKeys,
		}
		validated, err := validator.Validate()
		if err != nil {
			return nil, err
		}
		rootValue[p] = validated
	}
	return rootValue, nil
}
```

- [ ] **Step 7: Run tests to verify they pass**

Run: `go test ./internal/patch/ -run TestOperationValidator_Validate_AllowNonScimKeys -v`
Expected: all PASS

- [ ] **Step 8: Run all patch tests for regressions**

Run: `go test ./internal/patch/ -v`
Expected: all PASS

- [ ] **Step 9: Commit**

```bash
git add internal/patch/patch.go internal/patch/update.go internal/patch/remove.go internal/patch/patch_test.go
git commit -m "feat: pass through non-SCIM attributes in PATCH when AllowNonScimKeys is set

getRefAttribute returns nil for unknown attributes instead of error.
validateUpdate/validateRemove return value as-is for nil refAttr.
validateEmptyPath passes unknown keys through without validation."
```

---

### Task 4: Wire `AllowNonScimKeys` from `ResourceType.validatePatch` to `patch.NewValidatorWithConfig`

**Files:**
- Modify: `resource_type.go:131-181` (validatePatch)
- Modify: `handlers_test.go`

- [ ] **Step 1: Write failing end-to-end test in `handlers_test.go`**

The test server already has `AllowNonScimKeys: true` on the User resource type (line 973). Add a test that sends PATCH with custom attributes:

```go
func TestServerPatchWithCustomAttributes(t *testing.T) {
	tests := []struct {
		name          string
		target        string
		body          string
		expectedCode  int
		expectedKey   string
		expectedValue interface{}
	}{
		{
			name:   "PATCH replace with string custom attribute",
			target: "/Users/0001",
			body: `{
				"schemas": ["urn:ietf:params:scim:api:messages:2.0:PatchOp"],
				"Operations": [{"op": "replace", "path": "custom_attribute", "value": "custom_value"}]
			}`,
			expectedCode:  http.StatusOK,
			expectedKey:   "custom_attribute",
			expectedValue: "custom_value",
		},
		{
			name:   "PATCH add with boolean custom attribute",
			target: "/Users/0001",
			body: `{
				"schemas": ["urn:ietf:params:scim:api:messages:2.0:PatchOp"],
				"Operations": [{"op": "add", "path": "custom_flag", "value": true}]
			}`,
			expectedCode:  http.StatusOK,
			expectedKey:   "custom_flag",
			expectedValue: true,
		},
		{
			name:   "PATCH add with empty path containing custom attribute",
			target: "/Users/0001",
			body: `{
				"schemas": ["urn:ietf:params:scim:api:messages:2.0:PatchOp"],
				"Operations": [{"op": "add", "value": {"userName": "updated", "custom_attribute": "custom_value"}}]
			}`,
			expectedCode:  http.StatusOK,
			expectedKey:   "custom_attribute",
			expectedValue: "custom_value",
		},
		{
			name:   "PATCH remove custom attribute",
			target: "/Users/0001",
			body: `{
				"schemas": ["urn:ietf:params:scim:api:messages:2.0:PatchOp"],
				"Operations": [{"op": "remove", "path": "custom_attribute"}]
			}`,
			expectedCode: http.StatusOK,
		},
	}

	for _, test := range tests {
		t.Run(test.name, func(t *testing.T) {
			req := httptest.NewRequest(http.MethodPatch, test.target, strings.NewReader(test.body))
			rr := httptest.NewRecorder()
			newTestServer(t).ServeHTTP(rr, req)

			assertEqualStatusCode(t, test.expectedCode, rr.Code)

			if test.expectedKey != "" && test.expectedValue != nil {
				var resource map[string]interface{}
				assertUnmarshalNoError(t, json.Unmarshal(rr.Body.Bytes(), &resource))
				assertEqual(t, test.expectedValue, resource[test.expectedKey])
			}
		})
	}
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `go test -run TestServerPatchWithCustomAttributes -v`
Expected: FAIL — HTTP 400 (invalid path)

- [ ] **Step 3: Modify `validatePatch` in `resource_type.go`**

Replace `patch.NewValidator` call with `patch.NewValidatorWithConfig`:

```go
func (t ResourceType) validatePatch(r *http.Request) ([]PatchOperation, *errors.ScimError) {
	data, err := readBody(r)
	if err != nil {
		return nil, &errors.ScimErrorInvalidSyntax
	}

	var req struct {
		Schemas    []string
		Operations []json.RawMessage
	}
	if err := unmarshal(data, &req); err != nil {
		return nil, &errors.ScimErrorInvalidSyntax
	}

	if len(req.Schemas) != 1 || req.Schemas[0] != "urn:ietf:params:scim:api:messages:2.0:PatchOp" {
		return nil, &errors.ScimErrorInvalidValue
	}

	if len(req.Operations) < 1 {
		return nil, &errors.ScimErrorInvalidValue
	}

	config := patch.ValidatorConfig{
		AllowNonScimKeys: t.AllowNonScimKeys,
	}

	var operations []PatchOperation
	for _, v := range req.Operations {
		validator, err := patch.NewValidatorWithConfig(
			v,
			t.schemaWithCommon(),
			config,
			t.getSchemaExtensions()...,
		)
		if err != nil {
			return nil, &errors.ScimErrorInvalidPath
		}
		value, err := validator.Validate()
		if err != nil {
			return nil, &errors.ScimErrorInvalidValue
		}
		operations = append(operations, PatchOperation{
			Op:    string(validator.Op),
			Path:  validator.Path,
			Value: value,
		})
	}

	return operations, nil
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `go test -run TestServerPatchWithCustomAttributes -v`
Expected: all PASS

- [ ] **Step 5: Run all tests for regressions**

Run: `go test ./... -cover`
Expected: all PASS

- [ ] **Step 6: Commit**

```bash
git add resource_type.go handlers_test.go
git commit -m "feat: wire AllowNonScimKeys into PATCH validation pipeline

ResourceType.validatePatch now passes AllowNonScimKeys to
patch.NewValidatorWithConfig, enabling PATCH operations on
non-standard attributes when the flag is set."
```

---

### Task 5: Update `Schema.ValidatePatchOperation` to support non-SCIM keys

**Files:**
- Modify: `schema/schema.go:80-113`
- Modify: `schema/schema_test.go`

Note: `resource_type.go:validatePatch` does NOT call `ValidatePatchOperation` — it uses `internal/patch` exclusively. But `ValidatePatchOperation` is a public API used by external consumers (scalekit calls it directly). Adding the flag here keeps the API consistent.

- [ ] **Step 1: Write failing test in `schema/schema_test.go`**

```go
func TestSchema_ValidatePatchOperationAllowNonScimKeys(t *testing.T) {
	s := Schema{
		ID:   "test:Schema",
		Name: optional.NewString("Test"),
		Attributes: []CoreAttribute{
			SimpleCoreAttribute(SimpleStringParams(StringParams{
				Name: "knownAttr",
			})),
		},
	}

	t.Run("Rejects unknown attr by default", func(t *testing.T) {
		err := s.ValidatePatchOperationValue("add", map[string]interface{}{
			"unknown_attr": "value",
		})
		if err == nil {
			t.Error("expected error for unknown attribute")
		}
	})

	t.Run("Accepts unknown attr with AllowNonScimKeys", func(t *testing.T) {
		err := s.ValidatePatchOperation("add", map[string]interface{}{
			"unknown_attr": "value",
		}, false, true)
		if err != nil {
			t.Errorf("unexpected error: %v", err)
		}
	})

	t.Run("Still validates known attrs with AllowNonScimKeys", func(t *testing.T) {
		err := s.ValidatePatchOperation("add", map[string]interface{}{
			"knownAttr": "valid_value",
		}, false, true)
		if err != nil {
			t.Errorf("unexpected error: %v", err)
		}
	})
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `go test ./schema/ -run TestSchema_ValidatePatchOperationAllowNonScimKeys -v`
Expected: compilation error — `ValidatePatchOperation` has wrong number of args

- [ ] **Step 3: Update `ValidatePatchOperation` signature in `schema/schema.go`**

Add variadic `allowNonScimKeys` parameter. Existing callers pass 3 args and continue to work unchanged:

```go
// ValidatePatchOperation validates an individual operation and its related value.
func (s Schema) ValidatePatchOperation(operation string, operationValue map[string]interface{}, isExtension bool, allowNonScimKeys ...bool) *errors.ScimError {
	allowNonScim := len(allowNonScimKeys) > 0 && allowNonScimKeys[0]
	for k, v := range operationValue {
		var attr *CoreAttribute
		var scimErr *errors.ScimError

		for _, attribute := range s.Attributes {
			if strings.EqualFold(attribute.name, k) {
				attr = &attribute
				break
			}
			if isExtension && strings.EqualFold(s.ID+":"+attribute.name, k) {
				attr = &attribute
				break
			}
		}

		if attr == nil {
			if allowNonScim {
				continue
			}
			return &errors.ScimErrorInvalidValue
		}
		if cannotBePatched(operation, *attr) {
			return &errors.ScimErrorInvalidValue
		}

		if operation != "remove" {
			_, scimErr = attr.validate(v)
		}

		if scimErr != nil {
			return scimErr
		}
	}

	return nil
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `go test ./schema/ -run TestSchema_ValidatePatchOperationAllowNonScimKeys -v`
Expected: all PASS

- [ ] **Step 5: Run all tests for regressions**

Run: `go test ./... -cover`
Expected: all PASS

- [ ] **Step 6: Commit**

```bash
git add schema/schema.go schema/schema_test.go
git commit -m "feat: add allowNonScimKeys parameter to Schema.ValidatePatchOperation

Uses variadic bool for backward compatibility. When set, unknown
attributes are skipped instead of rejected."
```

---

### Task 6: Final validation, lint, and push

- [ ] **Step 1: Run full test suite**

```bash
go test ./... -cover
```

Expected: all PASS

- [ ] **Step 2: Run linter**

```bash
go vet ./...
```

Expected: clean

- [ ] **Step 3: Run formatter**

```bash
go fmt ./...
```

Expected: no changes (or minor formatting fixes)

- [ ] **Step 4: Commit any lint/format fixes if needed, then push**

```bash
git push -u origin scim-patch-custom-attr
```

---

# Phase 2: Scalekit Cleanup

**Repo:** `~/Sandbox/scalekit3`
**Branch:** `scim-handler-patch` (off `origin/scim-handler`)

**Prerequisite:** Phase 1 merged/tagged in the `scalekit-inc/scim` fork. The scalekit go.mod currently has:
```
replace github.com/elimity-com/scim v0.0.0-20240320110924-172bf2aee9c8 => github.com/scalekit-inc/scim v1.0.0
```
This replace directive must point to a version/commit that includes the Phase 1 changes.

## File Map

| File | Action | What changes |
|------|--------|--------------|
| `go.mod` | Modify | Update `scalekit-inc/scim` version in replace directive |
| `directory/transfomer/request_transformer.go` | Modify | Remove 6 workaround functions + 2 code blocks |
| `directory/handlers/scim_handlers.go` | Modify | Remove enterprise extension rewriting in `Patch()` |
| `directory/transfomer/request_transformers_test.go` | Modify | Remove 5 workaround test cases + `TestEnterpriseExtensionPathValidation` |
| `directory/transfomer/testdata/TransformUnknownEnterprise*/` | Delete | 5 test data directories |

---

### Task 7: Create branch and update SCIM library dependency

- [ ] **Step 1: Create branch off scim-handler**

```bash
cd ~/Sandbox/scalekit3
git fetch origin
git checkout -b scim-handler-patch origin/scim-handler
```

- [ ] **Step 2: Update go.mod to point to the new SCIM library version**

Update the replace directive to point to the commit/tag that includes the Phase 1 changes. The exact version depends on how the fork is tagged:

```bash
# Update to point to the scim-patch-custom-attr branch or new tag
go get github.com/scalekit-inc/scim@<new-version-or-commit>
go mod tidy
```

- [ ] **Step 3: Verify the dependency compiles**

```bash
go build ./...
```

Expected: compiles successfully

- [ ] **Step 4: Commit**

```bash
git add go.mod go.sum
git commit -m "chore: update scim library with AllowNonScimKeys PATCH support"
```

---

### Task 8: Remove workaround functions from request_transformer.go

**Files:**
- Modify: `directory/transfomer/request_transformer.go`

**Remove these items completely:**
1. `enterpriseExtensionURI` constant
2. `knownEnterpriseAttributes` map
3. `isUnknownEnterpriseExtensionPath()` function
4. `marshalValue()` function
5. `rewriteEmptyPathEnterpriseExtension()` function
6. `transformEnterpriseExtensionBody()` function

**Remove these code blocks from existing functions:**

In `TransformPatch()`, remove the `isUnknownEnterpriseExtensionPath` case:
```go
// DELETE this entire case block:
case isUnknownEnterpriseExtensionPath(op.Path):
    attrName := op.Path[len(enterpriseExtensionURI)+1:]
    req.Operations[i] = PatchOperation{
        Op:   strings.ToLower(op.Op),
        Path: enterpriseExtensionURI + ":customAttributes",
        Value: []map[string]string{
            {"key": attrName, "value": marshalValue(op.Value)},
        },
    }
```

In `TransformPatch()`, remove `rewriteEmptyPathEnterpriseExtension` call from the empty-path case:
```go
// DELETE this line from the empty-path case:
rewriteEmptyPathEnterpriseExtension(&req.Operations[i], value)
```

In `TransformPayload()`, remove the `transformEnterpriseExtensionBody` block:
```go
// DELETE this entire block:
if r.Method != http.MethodPatch {
    attributes = transformEnterpriseExtensionBody(attributes)
}
```

Also remove the `"fmt"` import if it was only used by `marshalValue`.

**Keep everything else** (manager fix, string boolean conversion, nested attribute transforms, Okta roles, struct tags).

- [ ] **Step 1: Make the removals**
- [ ] **Step 2: Verify it compiles**

```bash
go build ./directory/...
```

- [ ] **Step 3: Commit**

```bash
git add directory/transfomer/request_transformer.go
git commit -m "refactor: remove enterprise extension attribute rewriting workaround

The SCIM library now supports AllowNonScimKeys for PATCH operations,
so unknown enterprise extension attributes pass through natively
without needing to be rewritten into customAttributes."
```

---

### Task 9: Simplify Patch handler in scim_handlers.go

**Files:**
- Modify: `directory/handlers/scim_handlers.go`

Remove the `transformedPatchOperations` block in `Patch()` that rewrites empty-path operations with enterprise extension URN prefixes. The SCIM library now passes these through directly.

**Before (remove):**
```go
transformedPatchOperations := make([]scim.PatchOperation, 0)

for _, operation := range operations {
    if operation.Path == nil {
        values, ok := operation.Value.(map[string]interface{})
        if !ok {
            continue
        }
        for key, value := range values {
            if strings.HasPrefix(key, "urn:ietf:params:scim:schemas:extension:enterprise:2.0") {
                patchOperation := scim.PatchOperation{
                    Op:    operation.Op,
                    Value: value,
                    Path: &filter.Path{
                        AttributePath: filter.AttributePath{
                            URIPrefix:     strToPtr("urn:ietf:params:scim:schemas:extension:enterprise:2.0:User"),
                            AttributeName: strings.TrimPrefix(key, "urn:ietf:params:scim:schemas:extension:enterprise:2.0:User:"),
                        },
                    },
                }
                transformedPatchOperations = append(transformedPatchOperations, patchOperation)
                delete(values, key)
            }
        }
    }
    transformedPatchOperations = append(transformedPatchOperations, operation)
}
```

**After:** Use `operations` directly instead of `transformedPatchOperations` in the loop below. Also remove unused imports (`strings`, `filter`) if they become unused.

- [ ] **Step 1: Make the simplification**
- [ ] **Step 2: Verify it compiles**

```bash
go build ./directory/...
```

- [ ] **Step 3: Commit**

```bash
git add directory/handlers/scim_handlers.go
git commit -m "refactor: simplify Patch handler by removing enterprise extension workaround

SCIM library now handles non-standard attributes natively via
AllowNonScimKeys, eliminating the need for manual operation rewriting."
```

---

### Task 10: Remove workaround test cases and test data

**Files:**
- Modify: `directory/transfomer/request_transformers_test.go`
- Delete: 5 test data directories

**Remove from test file:**
1. These 5 test cases from the `tests` slice:
   - `"TransformUnknownEnterpriseAttrPatch"`
   - `"TransformUnknownEnterpriseAttrPost"`
   - `"TransformUnknownEnterpriseAttrEmptyPath"`
   - `"TransformUnknownEnterpriseAttrNestedEmptyPath"`
   - `"TransformUnknownEnterpriseAttrArrayValue"`
2. The same 5 entries from the `testName` slice in `InitializeTestData`
3. The entire `TestEnterpriseExtensionPathValidation` function
4. Unused imports (`scimfilter`, `schema`) if they become unused after removal

**Delete test data directories:**
```bash
rm -rf directory/transfomer/testdata/TransformUnknownEnterpriseAttrPatch
rm -rf directory/transfomer/testdata/TransformUnknownEnterpriseAttrPost
rm -rf directory/transfomer/testdata/TransformUnknownEnterpriseAttrEmptyPath
rm -rf directory/transfomer/testdata/TransformUnknownEnterpriseAttrNestedEmptyPath
rm -rf directory/transfomer/testdata/TransformUnknownEnterpriseAttrArrayValue
```

- [ ] **Step 1: Remove test cases and delete test data**
- [ ] **Step 2: Run remaining tests to verify nothing is broken**

```bash
go test ./directory/transfomer/ -v
```

Expected: remaining tests PASS

- [ ] **Step 3: Commit**

```bash
git add -A directory/transfomer/
git commit -m "test: remove workaround test cases for enterprise extension rewriting

These tests validated the customAttributes rewriting that is no longer
needed with the SCIM library fix."
```

---

### Task 11: Final validation and push

- [ ] **Step 1: Run full test suite**

```bash
go test ./directory/... -v
```

Expected: all PASS

- [ ] **Step 2: Verify build**

```bash
go build ./...
```

Expected: clean

- [ ] **Step 3: Push**

```bash
git push -u origin scim-handler-patch
```
