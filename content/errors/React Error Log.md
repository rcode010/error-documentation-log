---
title: React Error Documentation Log
date: 2026-01-30
draft: false
tags:
slug: React
---

## Error #1 - [Unexpected Logout After Company Update]

Date Encountered: 17th Dec 2025  
Severity: High  
Environment: Development  
Status: Resolved

### Error Description

Users experienced unexpected logouts immediately after updating company information, despite having valid access tokens. The application flow was:

1. User successfully updates company details via the desktop client
    
2. Update operation completes successfully on the backend
    
3. User is immediately and unexpectedly logged out with no error message or warning
    
4. User experience severely impacted - confusion and frustration
    

The frontend axios interceptor, which was recently updated to queue pending refresh token requests, was triggering token refresh attempts during app reload. The backend consistently rejected these refresh attempts with a 401 status and "Invalid refresh token" message, forcing the client-side logout logic to execute.

## Stack/Technology

- Frontend: Electron desktop client with TypeScript, Axios, Zustand (useUserStore)
    
- Backend: Node.js/Express API with JWT authentication
    
- Authentication: JWT with separate access and refresh tokens (stored in Electron secure storage)
    
- Environment: Staging server (Render.com)
    

### Error Message/Logs

Backend console output:  
"Invalid signature" (from jwt.verify() error)  
  
API Response:  
POST https://solution-squad-backend-development.onrender.com/api/admin/refresh-token Status: 401 Unauthorized  
Body: { "success": false, "message": "Invalid refresh token" }  
  
Client console:  
Auth refresh failed: AxiosError  
Token refresh failed  
Logout triggered

### Root Cause

Primary Issue: Improper use of location.reload() causing race conditions and auth state destruction.

### Technical Explanation:

1. Page Reload Destroys In-Memory State
    

- The EditCompanyModal.tsx component was calling location.reload() immediately after the update request
    
- This caused the entire app to reload while the update request was still in flight
    
- All in-memory state, including the accessToken stored in Zustand, was destroyed
    

3. Auth State Corruption During Reload
    

typescript

-   // EditCompanyModal.tsx - PROBLEMATIC CODE
    
-    const handleUpdate = async (e: React.FormEvent) => {
    
-      // ...
    
-      await updateCompany(formDataToSend, company._id);
    
-      globalThis.location.reload(); // ❌ Destroys auth state
    

   };

- When the page reloaded, App.tsx remounted and called refreshAuth()
    
- The access token was gone from memory
    
- The refresh token was retrieved from Electron secure storage
    
- However, due to timing issues during reload, the token was either:
    

- Stale/corrupted from a previous session
    
- Not fully persisted from a recent refresh
    
- Racing with another token update operation
    

3. Token Verification Failure Chain
    

-   Time 0ms:   User clicks "Save Changes"
    
-    Time 10ms:  Update request sent
    
-    Time 20ms:  location.reload() called (doesn't wait for response)
    
-    Time 30ms:  Page starts reloading → Zustand state destroyed
    
-    Time 50ms:  App.tsx remounts
    
-    Time 60ms:  refreshAuth() called (no accessToken in memory)
    
-    Time 70ms:  Retrieves refresh token from secure storage
    
-    Time 80ms:  Token is invalid/corrupted/stale
    
-    Time 90ms:  Backend jwt.verify() fails with "Invalid signature"
    

   Time 100ms: axios interceptor catches 401 → logout()

4. Why the New Axios Interceptor Made It Visible
    

- Before the interceptor improvements, 401 errors might have silently failed
    
- The new interceptor properly handles 401s by attempting token refresh
    
- When refresh fails, it correctly logs the user out as a security measure
    
- This proper error handling exposed the underlying reload timing issue
    

Discovery process:

1. Initially suspected JWT secret mismatch between environments ❌
    
2. Traced API calls - request structure was correct
    
3. Added extensive logging to axios interceptor and refresh flow
    
4. Noticed the error occurred specifically after company updates
    
5. Discovered location.reload() was being called in modal handlers
    
6. Realized reload was destroying auth state mid-request
    
7. Found same anti-pattern in multiple components (CompaniesPage, EditCompanyModal, AddCompanyModal)
    

### How We Fixed It

### 1. Removed All location.reload() Calls

Replaced hard reloads with proper state management using Zustand's built-in reactivity.

### 2. Updated Store to Return Success/Failure Booleans

3. Updated Modal Handlers to Wait for Completion

### 4. Fixed Refresh Button in CompaniesPage

CompaniesPage.tsx:

typescript

const refresh = async (): Promise<void> => {

  setIsRefreshing(true);

  await getCompanies(); // ✅ Just refetch data, don't reload page

  setIsRefreshing(false);

};

  

### Prevention/Lessons Learned

### Key Takeaways:

1. Never use location.reload() in modern React applications
    

- Destroys all in-memory state (auth tokens, form data, UI state)
    
- Creates race conditions with in-flight API requests
    
- Breaks single-page application architecture
    
- Provides poor user experience (flash, loss of scroll position)
    

3. Proper State Management Practices:
    

- Use Zustand/Redux reactivity for data updates
    
- Store actions should call get().getCompanies() to refresh data naturally
    
- Components automatically re-render when store data changes
    
- No need for manual page reloads
    

5. Async Operation Handling:
    

- Always await API calls before taking UI actions
    
- Return success/failure booleans from store actions
    
- Close modals/dialogs only AFTER operations complete
    
- Show loading states during operations
    

7. Authentication State Protection:
    

- Never destroy auth state unnecessarily
    
- Keep access tokens in memory (Zustand)
    
- Keep refresh tokens in secure storage (Electron secure storage)
    
- Rely on axios interceptors for token refresh, not manual reloads
    

9. Error Handling Best Practices:
    

- Proper 401 handling reveals underlying issues (which is good!)
    
- Silent failures hide bugs - always handle errors explicitly
    
- Log errors comprehensively for debugging
    
- Toast notifications for user feedback
    

  

### Code Review Checklist:

- No location.reload() or window.location.reload() calls
    
- All async operations are properly awaited
    
- Store actions return success/failure status
    
- Modal handlers wait for operations before closing
    
- Auth state is never destroyed during normal operations
    
- Loading states are shown during async operations
    

  

### Related Issues

N/A

**