# Integration Test Plan
## OMOP Microservices Pipeline - Comprehensive Testing

**Version:** 1.1  

---

## 1. Executive Summary

### 1.1 Purpose
This test plan ensures **reliable communication and data integrity** across all microservices in the OMOP entity extraction pipeline.

### 1.2 Testing Requirements

As specified by the project requirements, we must ensure:

1. ✅ **Services talk to each other reliably** (interfaces/contracts correct)
2. ✅ **Data does not get corrupted or lost** at any stage of pipeline execution
3. ✅ **Functional correctness**: entities & mappings are correct
4. ✅ **Non-functional**: performance, robustness, observability, security, and de-id quality

### 1.3 Scope

**In Scope:**
- ✅ Service-to-service communication (interfaces/contracts)
- ✅ Data integrity validation (no loss/corruption)
- ✅ Functional correctness (entity extraction, OMOP mappings)
- ✅ Performance testing (throughput, latency)
- ✅ Robustness testing (error handling, recovery)
- ✅ Observability testing (logging, monitoring)

---

## 2. System Overview

### 2.1 Architecture

```
Raw Text Input
      ↓
[Preprocessing Service] → preprocessing-input-queue
      ↓
[Chunking Service] → chunking-input-queue
      ↓
[Entity Extraction Service] → entity-extraction-input-queue
      ↓
[Standardization Service] → standardization-input-queue
      ↓
Final Output (sequencer-input-queue)
```

### 2.2 Message Flow
- Each service consumes from its input queue
- Processes the message
- Publishes result to the next service's input queue
- Uses RabbitMQ for asynchronous messaging

---

## 3. Test Categories 

### 3.1 Requirement Coverage Matrix

| Requirement | Test Category | Test IDs | Status |
|-------------|---------------|----------|--------|
| **1. Reliable Communication** | Contract Validation | TC-101 to TC-104 | ✅ Covered |
| **2. No Data Loss/Corruption** | Data Integrity | TC-201 to TC-205 | ✅ Covered |
| **3. Functional Correctness** | Functional Validation | TC-301 to TC-305 | ✅ Covered |
| **4. Performance** | Performance Testing | TC-401 to TC-403 | ✅ Covered |
| **5. Robustness** | Error Handling | TC-501 to TC-505 | ✅ Covered |
| **6. Observability** | Logging/Monitoring | TC-601 to TC-603 | ✅ Covered |
| **7. Security** | Security Testing | TC-701 | ⚠️ Future |
| **8. De-ID Quality** | De-ID Validation | TC-801 | ⚠️ Future |

**Total Tests: 25 (23 current + 2 future)**

---

## 4. Contract Specifications

### 4.1 MessageStatus Enum

All services use the following standard message status values:

```python
class MessageStatus(str, Enum):
    PENDING = "pending"        # Message queued, not yet processed
    PROCESSING = "processing"  # Currently being processed
    COMPLETED = "completed"    # Successfully processed
    FAILED = "failed"         # Processing failed with error
```

**Usage Rules:**
- Initial messages start with `status="pending"`
- Services set `status="processing"` when starting work
- Successful completion → `status="completed"`
- Any error → `status="failed"` with error details in metadata

---

### 4.2 Common Fields

All messages across the pipeline MUST include these common fields:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `message_id` | string (UUID) | Yes | Unique identifier for this message |
| `timestamp` | string (ISO 8601) | Yes | UTC timestamp in format: `YYYY-MM-DDTHH:MM:SS.sssZ` |
| `correlation_id` | string (UUID) | Yes | Links all messages in same pipeline run |
| `status` | MessageStatus | Yes | Current processing status |
| `metadata` | dict | Yes | Contains `patient_id`, `document_id`, and processing info |

**Validation Rules:**
- ✅ `message_id` must be unique UUID v4
- ✅ `timestamp` must be valid UTC ISO 8601 format
- ✅ `correlation_id` must remain unchanged throughout pipeline
- ✅ `status` must be one of the enum values
- ✅ `metadata.patient_id` and `metadata.document_id` must never be null/empty

---

### 4.3 Contract Validation Checklist

Before any service accepts a message, it MUST validate:

**✅ Structure Validation:**
- [ ] All required fields present (no missing keys)
- [ ] Field types match specification (string, dict, list, etc.)
- [ ] Nested objects have correct structure

**✅ Enum Validation:**
- [ ] `status` is one of: pending, processing, completed, failed
- [ ] `domain` (if present) is valid OMOP domain

**✅ Metadata Validation:**
- [ ] `metadata` is a dict (not null)
- [ ] `metadata.patient_id` exists and non-empty
- [ ] `metadata.document_id` exists and non-empty
- [ ] `metadata.correlation_id` matches top-level `correlation_id`

**✅ Timestamp Validation:**
- [ ] `timestamp` in UTC ISO 8601 format
- [ ] Timestamp not in the future
- [ ] Timestamp reasonable (within last 24 hours for real-time processing)

**✅ Content Validation:**
- [ ] Text fields non-empty (when status=completed)
- [ ] Lists have correct item types
- [ ] No SQL injection or malicious content

---

## 5. Test Cases

## REQUIREMENT 1: RELIABLE COMMUNICATION (Interfaces/Contracts)

### TC-101: Preprocessing → Chunking Contract Validation ✅

**Objective:** Verify Preprocessing output conforms to Chunking input contract

**Priority:** Critical

**Test Steps:**
1. Publish valid message to `preprocessing-input-queue`
2. Consume output from `chunking-input-queue`
3. Validate message structure against contract

**Input Message:**
```json
{
  "message_id": "msg-001",
  "correlation_id": "corr-001",
  "timestamp": "2024-11-25T10:30:00.000Z",
  "raw_text": "Patient presents with chest pain...",
  "status": "pending",
  "metadata": {
    "patient_id": "TEST001",
    "document_id": "DOC001"
  }
}
```

**Validation Checklist:**
- ✅ All required fields present: `message_id`, `timestamp`, `preprocessed_text`, `status`, `metadata`
- ✅ Field types correct: `preprocessed_text` is string, `status` is MessageStatus enum
- ✅ Enums valid: `status` in ["pending", "processing", "completed", "failed"]
- ✅ Metadata contains: `patient_id`, `document_id`, `correlation_id`
- ✅ Timestamp in UTC ISO format: `YYYY-MM-DDTHH:MM:SS.sssZ`
- ✅ `preprocessed_text` not empty (if status=completed)
- ✅ `correlation_id` unchanged from input

**Expected Output Schema:**
```json
{
  "message_id": "msg-002",
  "correlation_id": "corr-001",
  "timestamp": "2024-11-25T10:30:02.150Z",
  "preprocessed_text": "patient presents with chest pain",
  "status": "completed",
  "metadata": {
    "patient_id": "TEST001",
    "document_id": "DOC001",
    "preprocessing_stats": {
      "original_length": 45,
      "cleaned_length": 38
    }
  }
}
```

**Expected Result:** ✓ Contract validated, Chunking can consume message

---

### TC-102: Chunking → Entity Extraction Contract Validation ✅

**Objective:** Verify Chunking output conforms to Entity Extraction input contract

**Priority:** Critical

**Validation Checklist:**
- ✅ Required fields: `message_id`, `timestamp`, `chunks`, `chunk_count`, `status`, `metadata`
- ✅ `chunks` is List[Chunk] with correct structure
- ✅ `chunk_count` matches actual number of chunks
- ✅ Each chunk valid: `chunk_id`, `text`, `start_pos`, `end_pos`, `word_count`, `char_count`
- ✅ Chunk positions are sequential and non-overlapping
- ✅ Metadata preserved and extended with chunking stats
- ✅ `correlation_id` unchanged

**Expected Output Schema:**
```json
{
  "message_id": "msg-003",
  "correlation_id": "corr-001",
  "timestamp": "2024-11-25T10:30:05.300Z",
  "chunks": [
    {
      "chunk_id": "chunk-001",
      "text": "patient presents with chest pain",
      "start_pos": 0,
      "end_pos": 32,
      "word_count": 5,
      "char_count": 32
    }
  ],
  "chunk_count": 1,
  "status": "completed",
  "metadata": {
    "patient_id": "TEST001",
    "document_id": "DOC001"
  }
}
```

**Expected Result:** ✓ Contract validated, Entity Extraction can consume

---

### TC-103: Entity Extraction → Standardization Contract Validation ✅

**Objective:** Verify Entity Extraction output conforms to Standardization input contract

**Priority:** Critical

**Validation Checklist:**
- ✅ Required fields: `message_id`, `timestamp`, `terms`, `metadata`
- ✅ `terms` is List[TermToStandardize], minimum 0 items (empty if no entities found)
- ✅ Each term valid: `term`, `domain`, `entity_id`
- ✅ `domain` in allowed list: Drug, Measurement, Procedure, Condition, Observation, Device, Specimen
- ✅ `term` non-empty string
- ✅ `entity_id` unique within message
- ✅ Metadata preserved

**Expected Output Schema:**
```json
{
  "message_id": "msg-004",
  "correlation_id": "corr-001",
  "timestamp": "2024-11-25T10:30:35.450Z",
  "terms": [
    {
      "term": "chest pain",
      "domain": "Condition",
      "entity_id": "entity-001"
    },
    {
      "term": "aspirin",
      "domain": "Drug",
      "entity_id": "entity-002"
    }
  ],
  "status": "completed",
  "metadata": {
    "patient_id": "TEST001",
    "document_id": "DOC001"
  }
}
```

**Expected Result:** ✓ Contract validated, Standardization can consume

---

### TC-104: Standardization Output Contract Validation ✅

**Objective:** Verify Standardization produces valid final output

**Priority:** Critical

**Validation Checklist:**
- ✅ Required fields: `message_id`, `timestamp`, `results`, `status`, `metadata`
- ✅ `results` is List[StandardizedTerm]
- ✅ Each result valid: `original_term`, `domain`, `entity_id`, `match_type`, `matches`
- ✅ `match_type` in: direct, ingredient, no_match
- ✅ `matches` is list of concept codes (can be empty if no_match)
- ✅ All original entity_ids accounted for

**Expected Output Schema:**
```json
{
  "message_id": "msg-005",
  "correlation_id": "corr-001",
  "timestamp": "2024-11-25T10:31:15.780Z",
  "results": [
    {
      "original_term": "chest pain",
      "domain": "Condition",
      "entity_id": "entity-001",
      "match_type": "direct",
      "matches": [
        {
          "concept_id": 29555009,
          "concept_name": "Chest pain",
          "vocabulary_id": "SNOMED"
        }
      ]
    }
  ],
  "status": "completed",
  "metadata": {
    "patient_id": "TEST001",
    "document_id": "DOC001"
  }
}
```

**Expected Result:** ✓ Output conforms to contract, ready for downstream consumption

---

## REQUIREMENT 2: NO DATA LOSS/CORRUPTION

### TC-201: Metadata Preservation ✅

**Objective:** Verify patient_id and document_id never lost or changed

**Priority:** Critical

**Test Steps:**
1. Publish message with `patient_id="TEST001"`, `document_id="DOC001"`
2. Track through all 4 services
3. Verify unchanged at each stage

**Validation at Each Stage:**
- ✅ Preprocessing output: patient_id=TEST001, document_id=DOC001
- ✅ Chunking output: patient_id=TEST001, document_id=DOC001
- ✅ Entity Extraction output: patient_id=TEST001, document_id=DOC001
- ✅ Standardization output: patient_id=TEST001, document_id=DOC001

**Edge Cases to Test:**
- ✅ Special characters in IDs: `patient_id="TEST-001_#2"`
- ✅ Very long IDs (>100 chars)
- ✅ Unicode in metadata values

**Expected Result:** ✓ Metadata never lost, modified, or corrupted

---

### TC-202: Correlation ID Tracking ✅

**Objective:** Verify correlation_id maintained for end-to-end traceability

**Priority:** Critical

**Test Steps:**
1. Generate unique correlation_id: `corr-test-12345`
2. Publish message with this correlation_id
3. Verify same correlation_id in all intermediate outputs
4. Query logs by correlation_id to trace full pipeline

**Validation:**
- ✅ Correlation ID in preprocessing output
- ✅ Correlation ID in chunking output
- ✅ Correlation ID in entity extraction output
- ✅ Correlation ID in standardization output
- ✅ All logs tagged with same correlation_id
- ✅ Can reconstruct full message flow from logs

**Expected Result:** ✓ Correlation ID enables full pipeline tracing

---

### TC-203: Message Count Validation ✅

**Objective:** Verify no messages lost in pipeline

**Priority:** Critical

**Test Steps:**
1. Publish 100 messages with unique IDs: `msg-001` to `msg-100`
2. Track each message through pipeline
3. Verify 100 outputs received (with completed or failed status)
4. Wait for processing to stabilize (queues empty)

**Validation:**
- ✅ Input count = Output count (100 in, 100 out)
- ✅ No duplicate processing (each message_id appears once)
- ✅ All message_ids accounted for
- ✅ No messages stuck in queues
- ✅ Success + Failure counts = Total count

**Success Criteria:**
- Zero message loss
- 100% accountability

**Expected Result:** ✓ Zero message loss confirmed

---

### TC-204: Data Content Integrity ✅

**Objective:** Verify text content not corrupted during processing

**Priority:** High

**Test Cases:**

**Test 1: Special Characters**
- Input: `"Patient has 50mg/dL glucose & <10% risk"`
- Verify: `&`, `<`, `>`, `/` preserved correctly

**Test 2: Unicode Characters**
- Input: `"Patient name: José García, diagnosis: café-au-lait spots"`
- Verify: Accented characters preserved

**Test 3: Medical Symbols**
- Input: `"Temperature 98.6°F, BP 120/80 mmHg, O₂ sat 95%"`
- Verify: Degree symbol, subscripts preserved

**Test 4: Newlines and Whitespace**
- Input: Multi-line text with `\n`, `\t`, multiple spaces
- Verify: Formatting preserved or normalized consistently

**Test 5: Maximum Length**
- Input: 50,000 character clinical note
- Verify: No truncation, complete text preserved

**Validation:**
- ✅ No data truncation
- ✅ Special characters handled correctly
- ✅ Unicode preserved (UTF-8 encoding)
- ✅ No encoding corruption (Latin-1, ASCII issues)
- ✅ Whitespace normalized consistently

**Expected Result:** ✓ Content integrity maintained through pipeline

---

### TC-205: Chunk Completeness ✅

**Objective:** Verify chunking doesn't lose text portions

**Priority:** High

**Test Steps:**
1. Publish text of known length: 5000 characters
2. Retrieve all chunks from chunking output
3. Concatenate all chunk texts
4. Verify completeness

**Validation:**
- ✅ Sum of chunk char_count ≈ original text length (±5% for cleaning)
- ✅ No text portions missing (check for known phrases)
- ✅ Chunk positions (start_pos, end_pos) are contiguous
- ✅ No overlapping chunks
- ✅ No gaps between chunks (end_pos[n] ≈ start_pos[n+1])
- ✅ chunk_count matches len(chunks)

**Test Data:**
```
"Patient History: [500 words]
Current Medications: [300 words]
Lab Results: [400 words]
Assessment: [200 words]
Plan: [150 words]"
```

**Verify all sections present in output chunks**

**Expected Result:** ✓ No text lost during chunking, complete coverage

---

## REQUIREMENT 3: FUNCTIONAL CORRECTNESS

### TC-301: Entity Extraction Validation 

**Objective:** Verify entities extracted correctly from clinical text

**Test Data:** Clinical note with known entities:
- Drugs: "Aspirin 325mg", "Metoprolol 50mg"
- Conditions: "Myocardial Infarction", "Hypertension"
- Procedures: "Cardiac Catheterization"
- Measurements: "Complete Blood Count"

**Validation:**
- ✅ All expected entities extracted
- ✅ Entity text matches source
- ✅ No hallucinated entities
- ✅ Entity boundaries correct

**Expected Result:** ✓ Entities correctly identified

---

### TC-302: OMOP Domain Mapping Validation 

**Objective:** Verify entities mapped to correct OMOP domains

**Validation:**
- ✅ "Aspirin" → Drug domain
- ✅ "Complete Blood Count" → Measurement domain
- ✅ "Cardiac Catheterization" → Procedure domain
- ✅ "Myocardial Infarction" → Condition domain
- ✅ No incorrect domain assignments

**Expected Result:** ✓ Domain mappings functionally correct

---

### TC-303: Standardization Accuracy

**Objective:** Verify terms standardized to correct codes

**Test Cases:**
- "Aspirin 325mg" → Should match RxCUI in RxNorm
- "Complete Blood Count" → Should match LOINC code
- "Myocardial Infarction" → Should match SNOMED code

**Validation:**
- ✅ Direct matches found for common terms
- ✅ Ingredient matches for drugs
- ✅ Match_type correctly assigned
- ✅ No false matches (high confidence)

**Expected Result:** ✓ Standardization functionally accurate

---

### TC-304: End-to-End Functional Validation 

**Objective:** Verify complete pipeline produces correct output

**Test Steps:**
1. Use realistic clinical discharge summary
2. Manually identify expected entities
3. Compare pipeline output to expected

**Validation:**
- ✅ Recall: % of expected entities found
- ✅ Precision: % of extracted entities correct
- ✅ F1-score acceptable (>80%)

**Expected Result:** ✓ Pipeline functionally correct

---

### TC-305: Edge Case Handling

**Objective:** Verify functional correctness for edge cases

**Test Cases:**
- Very short text (< 50 words)
- Very long text (> 10,000 words)
- Text with no medical entities
- Text with only medications
- Text with misspellings

**Expected Result:** ✓ Pipeline handles edge cases gracefully

---

## REQUIREMENT 4: PERFORMANCE

### TC-401: Pipeline Throughput 

**Objective:** Measure documents processed per minute

**Test Steps:**
1. Publish 50 clinical notes
2. Measure time from first publish to last output
3. Calculate throughput

**Target Performance:**
- Minimum: 5 documents/minute
- Target: 10 documents/minute

**Expected Result:** ✓ Throughput meets target

---

### TC-402: Service Latency

**Objective:** Measure per-service processing time

**Test Steps:**
1. Publish timestamped messages
2. Measure time at each queue
3. Calculate per-service latency

**Target Latencies:**
- Preprocessing: < 2 seconds
- Chunking: < 5 seconds
- Entity Extraction: < 30 seconds
- Standardization: < 60 seconds
- Total: < 120 seconds per document

**Expected Result:** ✓ Latencies within acceptable range

---

### TC-403: Queue Backlog Monitoring

**Objective:** Verify queues don't accumulate unbounded messages

**Test Steps:**
1. Monitor queue depths during processing
2. Verify messages consumed faster than produced

**Validation:**
- ✅ Queue depths trend toward zero
- ✅ No indefinite accumulation
- ✅ Services keep up with input rate

**Expected Result:** ✓ No performance bottlenecks

---

## REQUIREMENT 5: ROBUSTNESS

### TC-501: Service Crash Recovery 

**Objective:** Verify pipeline recovers from service failures

**Test Steps:**
1. Start processing 10 messages
2. Crash one service (kill process)
3. Restart service
4. Verify processing resumes

**Validation:**
- ✅ Messages requeued (NACK behavior)
- ✅ Service reconnects to RabbitMQ
- ✅ Processing continues from failure point
- ✅ No messages lost

**Expected Result:** ✓ Robust recovery from crashes

---

### TC-502: Failed Status Propagation

**Objective:** Verify failed messages don't break pipeline

**Test Steps:**
1. Inject message that causes preprocessing failure
2. Verify failed status propagates
3. Verify downstream services handle gracefully

**Validation:**
- ✅ Failed message published (not dropped)
- ✅ status="failed", error field populated
- ✅ Downstream services skip processing
- ✅ Pipeline continues for other messages

**Expected Result:** ✓ Robust error handling

---

### TC-503: Invalid Message Handling

**Objective:** Verify services handle malformed messages

**Test Steps:**
1. Publish message with invalid schema
2. Verify service logs error
3. Verify message NACK'd (requeued)
4. Verify service continues running

**Expected Result:** ✓ Services don't crash on bad input

---

### TC-504: Network Interruption Recovery

**Objective:** Verify services recover from RabbitMQ disconnection

**Test Steps:**
1. Start processing
2. Stop RabbitMQ briefly (docker-compose stop)
3. Restart RabbitMQ
4. Verify services reconnect automatically

**Expected Result:** ✓ Automatic reconnection works

---

## REQUIREMENT 6: OBSERVABILITY

### TC-601: Logging Completeness 

**Objective:** Verify all critical events logged

**Validation:**
- ✅ Message received (with message_id)
- ✅ Processing started
- ✅ Processing completed (with status)
- ✅ Errors logged (with details)
- ✅ Message published to next queue
- ✅ Correlation_id in all logs

**Expected Result:** ✓ Complete audit trail via logs

---

### TC-602: Error Traceability

**Objective:** Verify errors can be traced to source

**Test Steps:**
1. Inject error (empty text)
2. Check logs across all services
3. Verify correlation_id enables tracing

**Validation:**
- ✅ Error logged in preprocessing
- ✅ Failed status logged in chunking
- ✅ Same correlation_id in all logs
- ✅ Can reconstruct error flow

**Expected Result:** ✓ Full error traceability

---

### TC-603: Metrics Availability

**Objective:** Verify key metrics can be collected

**Metrics to Check:**
- ✅ Messages processed per service
- ✅ Processing time per service
- ✅ Success/failure rates
- ✅ Queue depths
- ✅ Error counts by type

**Expected Result:** ✓ Metrics available for monitoring

---

## REQUIREMENT 7: SECURITY (Future Phase)

### TC-701: Data Security (Future)

**Note:** To be implemented in future phase

**Will Cover:**
- Message encryption in transit
- Access control to queues
- Sensitive data handling (PHI/PII)
- Audit logging of data access

---

## REQUIREMENT 8: DE-ID QUALITY (Future Phase)

### TC-801: De-identification Quality (Future)

**Note:** To be implemented in future phase

---

## 5. Test Execution Plan

### 5.1 Folder Structure

```
MBAI/
├── integration-tests/
│   ├── test_contracts.py          # TC-101 to TC-104 (Communication)
│   ├── test_data_integrity.py     # TC-201 to TC-205 (No data loss)
│   ├── test_functional.py         # TC-301 to TC-305 (Correctness)
│   ├── test_performance.py        # TC-401 to TC-403 (Performance)
│   ├── test_robustness.py         # TC-501 to TC-504 (Robustness)
│   ├── test_observability.py      # TC-601 to TC-603 (Logging)
│   ├── conftest.py                # Pytest fixtures
│   ├── requirements.txt           # Dependencies
│   └── README.md                  # Execution guide
├── services/
├── messaging/
├── e2e-tests/
└── infrastructure/
```


## 6. Requirements Coverage Summary

### ✅ **FULLY COVERED:**

**1. Reliable Communication (Interfaces/Contracts)**
- TC-101 to TC-104: All service contracts validated

**2. No Data Loss/Corruption**
- TC-201 to TC-205: Metadata, correlation IDs, message counts, content integrity

**3. Functional Correctness**
- TC-301 to TC-305: Entity extraction, domain mapping, standardization accuracy

**4. Performance**
- TC-401 to TC-403: Throughput, latency, queue monitoring

**5. Robustness**
- TC-501 to TC-504: Crash recovery, error handling, network recovery

**6. Observability**
- TC-601 to TC-603: Logging, tracing, metrics

###  **FUTURE PHASE:**

**7. Security**
- TC-701: Requires security team involvement

**8. De-ID Quality**
- TC-801: Requires medical expert validation

---

## 7. Summary

**What We're Testing:**
- ✅ Reliable communication (4 tests)
- ✅ No data loss (5 tests)
- ✅ Functional correctness (5 tests)
- ✅ Performance (3 tests)
- ✅ Robustness (4 tests)
- ✅ Observability (3 tests)

**Total: 24 tests (23 current + 1 future)**

**Critical Tests:** 13 tests must pass (Communication + Data Integrity + Functional)

---

