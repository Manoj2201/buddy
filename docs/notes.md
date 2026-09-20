
# Topics to be Discussed

## Asset Upload API

### Endpoint
```
POST /asset/upload
```

**Description:** Upload any assets (image, pdf, etc.) to bucket and store reference in asset table.

### Request Format
```json
{
  "target_type": "user_profile",
  "target_id": "UUID-HASH-###",
  "type": ".png",
  "file_path": "c:user/abc.png"
}
```

### Upload Flow
```
local_file_path → S3 → URI → APP → updates request → response
```

### Response Format
```json
{
  "target_type": "user_profile",
  "target_id": "UUID-HASH-###",
  "type": ".png",
  "url": "AWS_S3:__asdasdsa_.png"
}
```

---

## Phase 2 - Features & Considerations

### 1. Reporting an Account
**Key Questions:**
- How to decide if a user should be blocked?
- Analytics design for tracking reports against a specific user
  
**Metrics to Track:**
- **Frequency:** How many times has the user been reported?
- **Recency:** When was the last report filed?
- **Reason:** What are the common reasons for reports?

### 2. Reporting Scope
**Discussion Point:** Reporting at App level vs Blocking at Friendship level
- Should reports affect the entire app experience or just specific relationships?
- What are the implications of each approach?

### 3. Muting Feature
- User ability to mute notifications or content from specific users
- Scope and implementation details TBD

### 4. AI-Companion
- Integration and functionality requirements
- Use cases and implementation approach

### 5. Heavy File Upload Handling
**Asynchronous Processing Flow:**
1. Job ID gets created upon upload initiation
2. User receives notification that background work is in progress
3. Other APIs continue to function without blocking
4. User notified upon completion

**Benefits:**
- Non-blocking architecture
- Better user experience
- Improved system performance 

