# EasySave API Documentation

## Overview

This document describes the backend API endpoints used by the EasySave frontend application. The API provides authentication, user management, and data block CRUD operations.

## Base URL

```
http://63.179.18.244/api/
```

**Note**: This is currently HTTP. For production, use HTTPS for security.

## Authentication

All data management endpoints require authentication via request headers:

```
RequesterUsername: <username>
RequesterAccessKey: <access_key_token>
```

These headers are automatically added by the `sendRequest()` function in main.js.

## Endpoints

### Authentication & User Management

#### 1. Login

Authenticates a user and returns an access key.

**Endpoint**: `/api/login`  
**Method**: `GET`  
**Authentication**: None required

**Request Parameters** (Query String):
| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| username | string | Yes | User's username |
| password | string | Yes | User's password |

**Example Request**:
```
GET /api/login?username=johndoe&password=mypassword123
```

**Success Response** (200-299):
```json
{
    "accessKey": "abc123xyz789..."
}
```

**Error Responses**:

**401 Unauthorized**:
- Invalid username or password
- Frontend displays: "Login unsuccessful. Please check username and password."

**Other Errors**:
- Generic error
- Frontend displays: "Login failed"

**Frontend Implementation**:
```javascript
// In login.js
xhr.open("GET", "http://63.179.18.244/api/login?username=" + data.username + "&password=" + data.password, true);
```

**Usage Notes**:
- Access key should be stored securely (currently in cookie)
- Access key used for subsequent authenticated requests
- Consider implementing token expiration and refresh

---

#### 2. Create User (Register)

Creates a new user account.

**Endpoint**: `/api/create_user`  
**Method**: `POST`  
**Authentication**: None required

**Request Parameters** (Query String):
| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| username | string | Yes | Desired username (must be unique) |
| password | string | Yes | User's password |
| email | string | Yes | User's email address |

**Example Request**:
```
POST /api/create_user?username=johndoe&password=mypassword123&email=john@example.com
```

**Success Response** (200-299):
```json
{
    "status": "success",
    "message": "User created successfully"
}
```
- Frontend redirects to `/login` on success

**Error Response**:
```json
{
    "detail": "Error message describing the issue"
}
```

**Common Error Cases**:
- Username already exists
- Invalid email format
- Password doesn't meet requirements
- Missing required fields

**Frontend Implementation**:
```javascript
// In register.js
xhr.open("POST", "http://63.179.18.244/api/create_user?username=" + data.username + "&password=" + data.password + "&email=" + data.email, true);
```

**Usage Notes**:
- User must login after registration (no auto-login)
- Email may be used for password recovery (not implemented in frontend)
- Consider adding email verification flow

---

### Data Block Management

All endpoints in this section require authentication headers.

#### 3. Get Blocks

Retrieves all data blocks for the authenticated user.

**Endpoint**: `/api/get_blocks`  
**Method**: `GET`  
**Authentication**: Required

**Request Headers**:
```
RequesterUsername: <username>
RequesterAccessKey: <access_key>
Content-Type: application/json;charset=UTF-8
```

**Request Parameters** (Query String):
| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| extendedIdentifier | string | Yes | Filter by identifier prefix (empty string "" for all blocks) |

**Example Request**:
```
GET /api/get_blocks?extendedIdentifier=
Headers:
  RequesterUsername: johndoe
  RequesterAccessKey: abc123xyz789...
```

**Success Response** (200-299):
```json
{
    "blockList": [
        {
            "identifier": "prefix.username.myblock1",
            "value": "This is the value of my first block"
        },
        {
            "identifier": "prefix.username.myblock2",
            "value": "Another block value"
        }
    ]
}
```

**Response Fields**:
- `blockList`: Array of block objects
- `identifier`: Full block identifier (includes prefix and username)
- `value`: The block's content/value

**Frontend Processing**:
```javascript
// In main.js refreshTable()
response.blockList.forEach(function(value) {
    // Trims "prefix.username." from identifier for display
    trimmedIdentifier = value.identifier.split(".").slice(2).join(".")
    // Displays trimmedIdentifier and value in table
});
```

**Identifier Structure**:
```
[system_prefix].[username].[user_defined_name]
```

Frontend displays only the `[user_defined_name]` portion.

**Error Response**:
- 401 Unauthorized: Invalid credentials
- 403 Forbidden: Access denied

**Usage Notes**:
- Empty `extendedIdentifier` returns all user's blocks
- Specific identifier can filter results (not used in current frontend)
- Identifier is case-sensitive

---

#### 4. Create Block

Creates a new data block.

**Endpoint**: `/api/create_block`  
**Method**: `POST`  
**Authentication**: Required

**Request Headers**:
```
RequesterUsername: <username>
RequesterAccessKey: <access_key>
Content-Type: application/json;charset=UTF-8
```

**Request Parameters** (Query String):
| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| extendedIdentifier | string | Yes | Block identifier (user-defined name) |
| value | string | Yes | Block value/content |

**Example Request**:
```
POST /api/create_block?extendedIdentifier=myblock1&value=This%20is%20my%20data
Headers:
  RequesterUsername: johndoe
  RequesterAccessKey: abc123xyz789...
```

**Success Response** (200-299):
```json
{
    "status": "success",
    "message": "Block created successfully"
}
```

**Error Responses**:

**409 Conflict**:
```json
{
    "detail": "Block with this identifier already exists"
}
```

**400 Bad Request**:
```json
{
    "detail": "Invalid identifier or value"
}
```

**Frontend Implementation**:
```javascript
// In main.js createNewValue()
sendRequest("POST", "create_block", {
    "extendedIdentifier": identifier, 
    "value": value
}, function(error, response) {
    // Handle response
});
```

**Frontend Validation**:
- Identifier cannot be empty (validated before API call)
- Red border shown if validation fails

**Usage Notes**:
- Identifier must be unique per user
- Backend may add prefix and username to identifier
- Value can contain any text including multiline
- Special characters are URL-encoded by frontend

---

#### 5. Update Block

Updates an existing block's value.

**Endpoint**: `/api/update_block`  
**Method**: `PATCH`  
**Authentication**: Required

**Request Headers**:
```
RequesterUsername: <username>
RequesterAccessKey: <access_key>
Content-Type: application/json;charset=UTF-8
```

**Request Parameters** (Query String):
| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| extendedIdentifier | string | Yes | Block identifier to update |
| value | string | Yes | New value for the block |

**Example Request**:
```
PATCH /api/update_block?extendedIdentifier=myblock1&value=Updated%20value
Headers:
  RequesterUsername: johndoe
  RequesterAccessKey: abc123xyz789...
```

**Success Response** (200-299):
```json
{
    "status": "success",
    "message": "Block updated successfully"
}
```

**Error Responses**:

**404 Not Found**:
```json
{
    "detail": "Block not found"
}
```

**403 Forbidden**:
```json
{
    "detail": "Not authorized to update this block"
}
```

**Frontend Implementation**:
```javascript
// In main.js saveValue()
sendRequest("PATCH", "update_block", {
    "extendedIdentifier": identifier, 
    "value": value
}, function(error, response) {
    // Show green on success, red on error
});
```

**Frontend Feedback**:
- Success: Save icon turns green for 1 second
- Error: Save icon turns red for 1 second

**Usage Notes**:
- Identifier must exist
- User must own the block
- Can update to empty string (if backend allows)
- Value is completely replaced (not merged)

---

#### 6. Delete Block

Deletes a data block.

**Endpoint**: `/api/delete_block`  
**Method**: `POST`  
**Authentication**: Required

**Request Headers**:
```
RequesterUsername: <username>
RequesterAccessKey: <access_key>
Content-Type: application/json;charset=UTF-8
```

**Request Parameters** (Query String):
| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| extendedIdentifier | string | Yes | Block identifier to delete |

**Example Request**:
```
POST /api/delete_block?extendedIdentifier=myblock1
Headers:
  RequesterUsername: johndoe
  RequesterAccessKey: abc123xyz789...
```

**Success Response** (200-299):
```json
{
    "status": "success",
    "message": "Block deleted successfully"
}
```

**Error Responses**:

**404 Not Found**:
```json
{
    "detail": "Block not found"
}
```

**403 Forbidden**:
```json
{
    "detail": "Not authorized to delete this block"
}
```

**Frontend Implementation**:
```javascript
// In main.js deleteBlock()
sendRequest("POST", "delete_block", {
    "extendedIdentifier": identifier
}, function(error, response) {
    if (error == null) {
        // Remove row from DOM
        target.parentElement.parentElement.remove()
    }
});
```

**Frontend Behavior**:
- Success: Row immediately removed from table
- Error: Trash icon turns red for 1 second

**Usage Notes**:
- Deletion is permanent (no undo in frontend)
- User must own the block
- No confirmation dialog (consider adding)
- Deleted blocks cannot be recovered

---

## Common Response Patterns

### Success Response Structure

Most successful operations return:
```json
{
    "status": "success",
    "message": "Operation completed successfully"
}
```

Or with data:
```json
{
    "accessKey": "...",
    // other fields
}
```

### Error Response Structure

Errors typically return:
```json
{
    "detail": "Description of the error"
}
```

### HTTP Status Codes

| Code | Meaning | Common Use |
|------|---------|------------|
| 200 | OK | Request successful |
| 201 | Created | Resource created successfully |
| 400 | Bad Request | Invalid parameters or data |
| 401 | Unauthorized | Authentication failed |
| 403 | Forbidden | Not authorized for this resource |
| 404 | Not Found | Resource doesn't exist |
| 409 | Conflict | Resource already exists |
| 500 | Internal Server Error | Server-side error |

---

## Request Flow

### Authentication Flow

```
1. User submits login form
   ↓
2. Frontend sends GET /api/login with credentials
   ↓
3. Backend validates credentials
   ↓
4. Backend returns accessKey
   ↓
5. Frontend stores username and accessKey in cookies
   ↓
6. Frontend redirects to main page
   ↓
7. All subsequent requests include RequesterUsername and RequesterAccessKey headers
```

### Data Operation Flow

```
1. Frontend checks for auth cookie
   ↓
2. If not present, redirect to /login
   ↓
3. If present, load main page
   ↓
4. Call GET /api/get_blocks to fetch data
   ↓
5. Render data in table
   ↓
6. User action triggers API call (create/update/delete)
   ↓
7. Backend processes request
   ↓
8. Frontend updates UI based on response
```

---

## Frontend Request Helper

The `sendRequest()` function in main.js handles all API requests:

```javascript
function sendRequest(method, endpoint, data, callback) {
    // Constructs full URL: http://63.179.18.244/api/ + endpoint
    var fullUrl = "http://63.179.18.244/api/" + endpoint;
    
    // Adds query parameters from data object
    for (const key in data) {
        if (data.hasOwnProperty(key)) {
            const value = encodeURIComponent(data[key]);
            fullUrl += (fullUrl.includes('?') ? '&' : '?') + key + '=' + value;
        }
    }
    
    // Creates XMLHttpRequest
    const xhr = new XMLHttpRequest();
    xhr.open(method, fullUrl, true);
    
    // Sets headers
    xhr.setRequestHeader("Content-Type", "application/json;charset=UTF-8");
    xhr.setRequestHeader("RequesterUsername", username);
    xhr.setRequestHeader("RequesterAccessKey", auth);
    xhr.responseType = "json";
    
    // Handles response
    xhr.onreadystatechange = function () {
        if (xhr.readyState !== 4) return;
        
        if (xhr.status >= 200 && xhr.status < 300) {
            callback(null, xhr.response);
        } else {
            callback(new Error("Request failed: " + xhr.status), null);
        }
    };
    
    xhr.send(null);
}
```

**Usage Example**:
```javascript
sendRequest("GET", "get_blocks", {"extendedIdentifier": ""}, function(error, response) {
    if (error == null) {
        console.log("Success:", response);
    } else {
        console.log("Error:", error);
    }
});
```

---

## Security Considerations

### Current Implementation

**Strengths**:
- Credentials sent in headers (not in URL for data operations)
- Access key token-based authentication
- URL encoding of parameters

**Weaknesses**:
- HTTP instead of HTTPS (credentials sent in plain text)
- Access key stored in non-HttpOnly cookie (vulnerable to XSS)
- No CSRF protection
- No rate limiting (client-side)
- No request signing or encryption
- Login credentials in URL query string (visible in logs)

### Recommendations

**High Priority**:
1. **Use HTTPS** for all communications
2. **Use HttpOnly cookies** for access tokens
3. **Implement CSRF tokens** for state-changing operations
4. **Use POST for login** with credentials in request body
5. **Add request signing** or additional authentication layer

**Medium Priority**:
1. **Implement rate limiting** on authentication endpoints
2. **Add token expiration** and refresh mechanism
3. **Log all authentication attempts** for security monitoring
4. **Implement account lockout** after failed attempts

**Low Priority**:
1. **Add request/response encryption** beyond HTTPS
2. **Implement two-factor authentication**
3. **Add IP whitelisting** for sensitive operations

---

## Error Handling Guide

### For Developers

**Frontend Error Handling Pattern**:
```javascript
sendRequest("METHOD", "endpoint", {params}, function(error, response) {
    if (error == null) {
        // Success path
        showSuccessFeedback();
    } else {
        // Error path
        showErrorFeedback();
        console.error("API Error:", error);
    }
});
```

**Common Error Scenarios**:

1. **Network Failure**:
   - User loses internet connection
   - Backend server is down
   - DNS resolution fails

2. **Authentication Failure**:
   - Invalid credentials
   - Expired access token
   - Missing authentication headers

3. **Resource Conflicts**:
   - Duplicate identifier on create
   - Resource doesn't exist on update/delete
   - Permission denied

4. **Validation Errors**:
   - Empty required fields
   - Invalid data format
   - Field length violations

### For Users

**Visual Feedback**:
- **Green icon**: Operation successful
- **Red icon**: Operation failed
- **Spinner**: Operation in progress
- **Alert dialog**: Critical errors (login failure)

**Common User-Facing Errors**:

| Error | User Message | Action |
|-------|-------------|---------|
| Login failed | "Login unsuccessful. Please check username and password." | Verify credentials |
| Network error | "Request failed" (in console) | Check internet connection |
| Create duplicate | Red save icon | Choose different identifier |
| Invalid identifier | Red border on input | Fill required field |

---

## Testing the API

### Using curl

**Login**:
```bash
curl -X GET "http://63.179.18.244/api/login?username=testuser&password=testpass"
```

**Create Block**:
```bash
curl -X POST "http://63.179.18.244/api/create_block?extendedIdentifier=test1&value=testvalue" \
  -H "RequesterUsername: testuser" \
  -H "RequesterAccessKey: your_access_key"
```

**Get Blocks**:
```bash
curl -X GET "http://63.179.18.244/api/get_blocks?extendedIdentifier=" \
  -H "RequesterUsername: testuser" \
  -H "RequesterAccessKey: your_access_key"
```

**Update Block**:
```bash
curl -X PATCH "http://63.179.18.244/api/update_block?extendedIdentifier=test1&value=newvalue" \
  -H "RequesterUsername: testuser" \
  -H "RequesterAccessKey: your_access_key"
```

**Delete Block**:
```bash
curl -X POST "http://63.179.18.244/api/delete_block?extendedIdentifier=test1" \
  -H "RequesterUsername: testuser" \
  -H "RequesterAccessKey: your_access_key"
```

### Using JavaScript (Browser Console)

```javascript
// Login
fetch('http://63.179.18.244/api/login?username=testuser&password=testpass')
  .then(r => r.json())
  .then(d => console.log(d));

// Get blocks (replace with actual credentials)
fetch('http://63.179.18.244/api/get_blocks?extendedIdentifier=', {
  headers: {
    'RequesterUsername': 'testuser',
    'RequesterAccessKey': 'your_access_key'
  }
})
  .then(r => r.json())
  .then(d => console.log(d));
```

---

## Rate Limiting

**Status**: Not currently implemented in frontend

**Recommendations**:
- Add rate limiting to prevent abuse
- Suggested limits:
  - Login: 5 attempts per minute per IP
  - Create user: 3 attempts per minute per IP
  - Data operations: 60 requests per minute per user

---

## API Versioning

**Current Version**: Unversioned

**Recommendation**: Add version to API path:
```
http://63.179.18.244/api/v1/login
```

**Benefits**:
- Allows backward compatibility
- Easier to deprecate old endpoints
- Clear documentation of API changes

---

## Future Enhancements

### Proposed New Endpoints

1. **GET /api/search_blocks**
   - Search blocks by value content
   - Parameters: query string, filters

2. **POST /api/export_blocks**
   - Export all blocks as JSON/CSV
   - Returns downloadable file

3. **POST /api/import_blocks**
   - Import blocks from file
   - Parameters: file upload

4. **GET /api/user_profile**
   - Get user information
   - Returns: email, creation date, usage stats

5. **PATCH /api/update_password**
   - Change user password
   - Parameters: old password, new password

6. **POST /api/forgot_password**
   - Initiate password reset
   - Parameters: email

### Proposed Enhancements

1. **Batch Operations**:
   - Create/update/delete multiple blocks in one request
   - Reduces network overhead

2. **Pagination**:
   - Add page and limit parameters to get_blocks
   - Improves performance for users with many blocks

3. **Filtering and Sorting**:
   - Filter blocks by creation date, size, etc.
   - Sort by name, date, etc.

4. **WebSocket Support**:
   - Real-time updates for collaborative editing
   - Push notifications for changes

5. **File Attachments**:
   - Store files/images in blocks
   - Return signed URLs for downloads

---

## Changelog

### Version 1.0 (Current)
- Initial API documentation
- Documented all 6 endpoints
- Added security recommendations
- Included testing examples
- Provided error handling guide

---

## Support

For API-related issues:
1. Check this documentation
2. Review browser console for errors
3. Check network tab for request/response details
4. Open an issue in the GitHub repository

## Related Documentation

- [Main README](README.md) - Project overview and setup
- [Detailed Code Documentation](DOCUMENTATION.md) - Frontend code details
