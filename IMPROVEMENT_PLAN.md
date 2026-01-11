# Afflux 0.1 - Comprehensive Improvement Plan

**Generated:** January 5, 2026
**Status:** Draft - Pending Implementation
**Priority:** Security fixes first, then stability, then optimization

---

## Table of Contents

1. [Executive Summary](#executive-summary)
2. [API Keys Requiring Immediate Rotation](#api-keys-requiring-immediate-rotation)
3. [Critical Security Issues](#critical-security-issues)
4. [Error Handling Improvements](#error-handling-improvements)
5. [Code Quality Improvements](#code-quality-improvements)
6. [Performance Optimizations](#performance-optimizations)
7. [Configuration Management](#configuration-management)
8. [Implementation Timeline](#implementation-timeline)

---

## Executive Summary

The Afflux 0.1 workflow system is a sophisticated AI-driven video replication platform. This review identified **7 exposed API keys**, **multiple silent failure patterns**, and several areas for optimization. The most urgent action is rotating all exposed credentials.

---

## API Keys Requiring Immediate Rotation

### Summary Table

| # | Service | Status | Priority |
|---|---------|--------|----------|
| 1 | Google/Gemini API | 🔴 EXPOSED | CRITICAL |
| 2 | Google Gemini TTS | 🔴 EXPOSED | CRITICAL |
| 3 | Fal.ai API | 🔴 EXPOSED | CRITICAL |
| 4 | Apify Token #1 | 🔴 EXPOSED | CRITICAL |
| 5 | Apify Token #2 | 🔴 EXPOSED | CRITICAL |
| 6 | Rendi FFmpeg API | 🟡 ENCODED | HIGH |
| 7 | Replicate API | 🟡 PLACEHOLDER | MEDIUM |

---

### Detailed API Key Locations

#### 1. Google/Gemini API Key
- **Key Value:** `REDACTED`
- **File:** `tool_cacheAndAnalyzeVideo.json`
- **Locations:**
  - Line 18: `youtubeApiKey` field in "Set YouTube URL & Keys" node
  - Line 24: `geminiApiKey` field in "Set YouTube URL & Keys" node
- **Used For:** YouTube Data API, Gemini video analysis
- **Rotation URL:** https://console.cloud.google.com/apis/credentials

#### 2. Google Gemini TTS API Key
- **Key Value:** `REDACTED`
- **File:** `tool_geminiTTs.json`
- **Location:** Line 54 in "Gemini TTS Request" node HTTP body
- **Used For:** Google Text-to-Speech API
- **Rotation URL:** https://console.cloud.google.com/apis/credentials

#### 3. Fal.ai API Key
- **Key Value:** `REDACTED`
- **Files & Locations:**
  - `tool_batch_pic2video.json` - Line 32 in "Setup API Key" node
  - `tool_batch_gemini_edit_image.json` - Line 187 in HTTP header configuration
- **Used For:** SeeDANCE image-to-video, Kling fallback, Fal image editing
- **Rotation URL:** https://fal.ai/dashboard/keys

#### 4. Apify Token #1 (YouTube Downloader)
- **Key Value:** `REDACTED`
- **File:** `tool_cacheAndAnalyzeVideo.json`
- **Location:** Line 119 in "crwalvideo" HTTP request node
- **Used For:** YouTube video downloading via Apify actor
- **Rotation URL:** https://console.apify.com/account/integrations

#### 5. Apify Token #2 (Shot Detection)
- **Key Value:** `REDACTED`
- **File:** `tool_cacheAndAnalyzeVideo.json`
- **Location:** Line 137 in "shotdeteandpic" HTTP request node
- **Used For:** FFmpeg shot detection and frame extraction via Apify
- **Rotation URL:** https://console.apify.com/account/integrations

#### 6. Rendi FFmpeg API Key (Base64 Encoded)
- **Key Value (encoded):** `REDACTED`
- **Key Value (decoded):** `REDACTED`
- **File:** `tool_rendi_ffmpeg.json`
- **Location:** Line 11 in "Edit Fields" node (`FFMPEG-API-KEY` field)
- **Additional Key:** Line 17 - encrypted `RENDI-API-KEY` value
- **Used For:** Cloud FFmpeg video processing
- **Rotation URL:** https://rendi.dev/dashboard

#### 7. Replicate API Token (Placeholder)
- **Key Value:** `r8_YOUR_REPLICATE_API_TOKEN_HERE`
- **File:** `tool_batch_replicate_pic2video.json`
- **Location:** Line 34 in "Setup Config" code node
- **Status:** Placeholder - needs real key before use
- **Rotation URL:** https://replicate.com/account/api-tokens

---

### Rotation Checklist

```
[ ] 1. Google Cloud Console - Rotate API key REDACTED
[ ] 2. Google Cloud Console - Rotate API key REDACTED
[ ] 3. Fal.ai Dashboard - Rotate key REDACTED
[ ] 4. Apify Console - Rotate token REDACTED
[ ] 5. Apify Console - Rotate token REDACTED
[ ] 6. Rendi Dashboard - Rotate FFmpeg API key
[ ] 7. Replicate - Generate new token for tool_batch_replicate_pic2video
```

---

## Critical Security Issues

### Issue S1: Hardcoded API Keys in Workflow JSON

**Severity:** CRITICAL
**Impact:** Anyone with file access can steal credentials
**Affected Files:** 5 workflow files

**Current State:**
```javascript
// Example from tool_cacheAndAnalyzeVideo.json
{
  "name": "geminiApiKey",
  "value": "REDACTED"  // EXPOSED!
}
```

**Recommended Fix:**
1. Create n8n credentials for each service
2. Reference credentials by ID in workflows
3. Remove all hardcoded keys from JSON files

```javascript
// After fix - reference credential by ID
{
  "credentials": {
    "googleGeminiApi": {
      "id": "xxxx-credential-id",
      "name": "Google Gemini API"
    }
  }
}
```

---

### Issue S2: Base64 Encoding is NOT Encryption

**Severity:** HIGH
**File:** `tool_rendi_ffmpeg.json`

The Rendi API key is base64 encoded, which provides zero security:

```bash
# Anyone can decode this
echo "REDACTED" | base64 -d
# Output: hKusiqLvmdnstKyt3abB:073112374234e11298db41cc
```

**Recommended Fix:** Use n8n's encrypted credential storage instead.

---

### Issue S3: Test Data Contains Real URLs

**Severity:** MEDIUM
**Files:** `tool_batch_pic2video.json`, `tool_batch_gemini_edit_image.json`

The `pinData` sections contain real S3 URLs that would be exposed if workflows are shared:

```json
"pinData": {
  "When Executed by Another Workflow": [{
    "json": {
      "images": [{
        "pictureUrl": "https://video-analyze-perframe.s3.us-east-2.amazonaws.com/..."
      }]
    }
  }]
}
```

**Recommended Fix:** Clear `pinData` before committing or sharing workflows.

---

## Error Handling Improvements

### Issue E1: Silent Failure Pattern

**Severity:** HIGH
**Affected Nodes:** Multiple HTTP request nodes

Many nodes use `"onError": "continueRegularOutput"` which masks failures:

| File | Node Name | Line |
|------|-----------|------|
| tool_batch_pic2video.json | Submit to SeeDANCE | 90 |
| tool_batch_pic2video.json | Poll SeeDANCE Status | 188 |
| tool_batch_gemini_edit_image.json | Gemini Edit Request | ~100 |
| tool_batch_replicate_pic2video.json | Submit to Replicate | 94 |

**Current Behavior:** API failures are silently ignored, producing incomplete results.

**Recommended Fix:**
1. Change to `"onError": "continueErrorOutput"` to route errors separately
2. Add error aggregation node to collect and report all failures
3. Implement threshold-based failure detection (fail if >50% items fail)

---

### Issue E2: Missing Timeout Configuration

**Severity:** MEDIUM
**Affected Files:** All HTTP request nodes

No timeout is specified for HTTP requests, which could hang indefinitely.

**Recommended Fix:** Add timeout to all HTTP request nodes:
```json
{
  "options": {
    "timeout": 120000
  }
}
```

---

### Issue E3: Incomplete Retry Logic

**Severity:** MEDIUM
**File:** `tool_batch_pic2video.json`

Current polling strategy:
- Wait 50s initial
- Poll every 15s
- Max 3 retries

**Problem:** Only ~80 seconds total wait time. SeeDANCE jobs can take 2-3 minutes.

**Recommended Fix:**
1. Increase max retries to 10
2. Implement exponential backoff (10s, 20s, 40s, etc.)
3. Add maximum total wait time of 5 minutes

---

### Issue E4: No Array Validation Before Operations

**Severity:** MEDIUM
**Multiple Files**

```javascript
// Current - crashes if tasks is undefined
const tasks = $input.first().json.tasks;
const completed = tasks.filter(t => t.status === 'COMPLETED');

// Should be
const input = $input.first();
if (!input?.json?.tasks || !Array.isArray(input.json.tasks)) {
  throw new Error('Expected tasks array not found');
}
```

---

## Code Quality Improvements

### Issue C1: Magic Numbers

**Severity:** LOW
**File:** `tool_batch_pic2video.json`

```javascript
if (duration < 2) duration = 2;
if (duration > 12) duration = 12;  // Magic numbers
```

**Recommended Fix:** Define constants at top of code node:
```javascript
const MIN_DURATION = 2;
const MAX_DURATION = 12;
const DEFAULT_DURATION = 5;
```

---

### Issue C2: Mixed Language Comments

**Severity:** LOW
**Affected Files:** All workflow files

Comments are in Chinese and English, reducing maintainability:

```javascript
// 拆分 images 数组为多个 item
// ★★★ 核心修复：处理 Gemini 的混合结果 ★★★
```

**Recommended Fix:** Standardize all comments to English.

---

### Issue C3: Inconsistent Error Messages

**Severity:** LOW
**Multiple Files**

Error messages vary in format and detail:
- `'Unknown Error'`
- `item.json.error?.message || ''`
- `'SeeDANCE submission failed'`

**Recommended Fix:** Create standardized error format:
```javascript
{
  code: 'SEEDANCE_SUBMIT_FAILED',
  message: 'Failed to submit image to SeeDANCE API',
  segment_index: 0,
  original_error: 'API rate limit exceeded',
  timestamp: new Date().toISOString()
}
```

---

### Issue C4: Duplicate Code Patterns

**Severity:** MEDIUM
**Files:** `tool_batch_pic2video.json`, `tool_batch_replicate_pic2video.json`

Polling logic is duplicated 3 times:
1. SeeDANCE polling
2. Kling polling
3. Replicate polling

**Recommended Fix:** Create reusable sub-workflow `tool_poll_api_status` with parameters:
- `poll_url`
- `auth_header`
- `success_status`
- `max_wait_seconds`

---

## Performance Optimizations

### Issue P1: Fixed Wait Times

**Severity:** MEDIUM
**File:** `tool_batch_pic2video.json`

```json
{ "amount": 50, "name": "Wait 50s Initial" }
```

**Problem:** Wastes time if job completes in 10 seconds.

**Recommended Fix:** Start with 10s, use exponential backoff.

---

### Issue P2: Sequential Processing Where Parallel is Possible

**Severity:** LOW
**File:** `tool_cacheAndAnalyzeVideo.json`

Video analysis and shot detection run sequentially but could partially overlap.

**Recommended Fix:** After cache check, run video download and Gemini analysis in parallel.

---

### Issue P3: Inefficient Deduplication

**Severity:** LOW
**File:** `tool_batch_pic2video.json`

Current deduplication iterates through results twice.

**Recommended Fix:** Deduplicate during aggregation using a Map keyed by segment_index.

---

## Configuration Management

### Issue CM1: Hardcoded Infrastructure Values

**Severity:** MEDIUM

| Value | File | Current |
|-------|------|---------|
| S3 Bucket | tool_batch_gemini_edit_image.json | `amz-bucket-oktale` |
| Google Sheet ID | tool_write2sheet.json | `1PgWHiyMztcMEiNAhLR0VeQ_-MGSrYVkAqOquCn96jSE` |
| Supabase Table | tool_cacheAndAnalyzeVideo.json | `yt_video_cache` |

**Recommended Fix:** Create a configuration sub-workflow or use n8n environment variables.

---

### Issue CM2: No Workflow Versioning

**Severity:** LOW

No version tracking for workflows. Rollback requires manual backup restoration.

**Recommended Fix:**
1. Add version field to workflow metadata
2. Maintain changelog in each workflow's sticky notes
3. Use git for workflow JSON version control

---

## Implementation Timeline

### Phase 1: Critical Security (Week 1)
- [ ] Rotate all 7 API keys
- [ ] Move credentials to n8n credential manager
- [ ] Remove hardcoded keys from workflow JSONs
- [ ] Clear pinData test data

### Phase 2: Error Handling (Week 2)
- [ ] Replace `continueRegularOutput` with proper error routing
- [ ] Add timeout configuration to all HTTP nodes
- [ ] Improve retry logic with exponential backoff
- [ ] Add input validation to workflow triggers

### Phase 3: Code Quality (Week 3)
- [ ] Standardize error message format
- [ ] Replace magic numbers with constants
- [ ] Translate comments to English
- [ ] Add array validation checks

### Phase 4: Performance (Week 4)
- [ ] Implement exponential backoff for polling
- [ ] Create reusable polling sub-workflow
- [ ] Optimize deduplication logic
- [ ] Add parallel processing where possible

### Phase 5: Configuration (Week 5)
- [ ] Parameterize infrastructure values
- [ ] Implement workflow versioning
- [ ] Add monitoring and alerting
- [ ] Document architecture decisions

---

## Appendix: File Reference

| File | Purpose | Priority Issues |
|------|---------|-----------------|
| Tool Use v1.7.json | Main orchestrator | None (references sub-workflows) |
| tool_cacheAndAnalyzeVideo.json | Video analysis | 4 exposed API keys |
| tool_batch_pic2video.json | Image-to-video | 1 exposed key, silent failures |
| tool_batch_replicate_pic2video.json | Alternative I2V | Placeholder key |
| tool_batch_gemini_edit_image.json | Image editing | 1 exposed key, silent failures |
| tool_geminiTTs.json | Text-to-speech | 1 exposed key |
| tool_rendi_ffmpeg.json | Video composition | Base64 encoded key |
| tool_write2sheet.json | Results logging | Hardcoded sheet ID |

---

*Document maintained by: Development Team*
*Last reviewed: January 5, 2026*
