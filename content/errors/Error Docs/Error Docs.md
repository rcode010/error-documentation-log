---
title: "Error Documentation Log"
date: 2026-01-30
draft: false
tags:
---

# Error Documentation Log

**Last Updated:** 13th Jan 2026

---

## Table of Contents

- [[#Error 1 - Unexpected Logout After Company Update|Error #1 - Unexpected Logout After Company Update]]
- [[#Error 2 - Docker daemon - Host EC2 kernel conflict|Error #2 - Docker daemon - Host EC2 kernel conflict]]
- [[#Error 3 - E-Commerce Project Postponed|Error #3 - E-Commerce Project Postponed Due to Undefined Business Requirements]]

---

## Error #1 - Unexpected Logout After Company Update

**Date Encountered:** 17th Dec 2025  
**Severity:** 🔴 High  
**Environment:** Development  
**Status:** ✅ Resolved

### Error Description

Users experienced unexpected logouts immediately after updating company information, despite having valid access tokens. The application flow was:

- User successfully updates company details via the desktop client
- Update operation completes successfully on the backend
- User is immediately and unexpectedly logged out with no error message or warning
- User experience severely impacted - confusion and frustration

The frontend axios interceptor, which was recently updated to queue pending refresh token requests, was triggering token refresh attempts during app reload. The backend consistently rejected these refresh attempts with a 401 status and "Invalid refresh token" message, forcing the client-side logout logic to execute.

### Stack/Technology

- **Frontend:** Electron desktop client with TypeScript, Axios, Zustand (useUserStore)
- **Backend:** Node.js/Express API with JWT authentication
- **Authentication:** JWT with separate access and refresh tokens (stored in Electron secure storage)
- **Environment:** Staging server (Render.com)

### Error Message/Logs

**Backend console output:**

```
"Invalid signature" (from jwt.verify() error)
```

**API Response:**

```
POST https://solution-squad-backend-development.onrender.com/api/admin/refresh-token
Status: 401 Unauthorized
Body: { "success": false, "message": "Invalid refresh token" }
```

**Client console:**

```
Auth refresh failed: AxiosError
Token refresh failed
Logout triggered
```

### Root Cause

**Primary Issue:** Improper use of `location.reload()` causing race conditions and auth state destruction.

#### Technical Explanation

##### 1. Page Reload Destroys In-Memory State

- The `EditCompanyModal.tsx` component was calling `location.reload()` immediately after the update request
- This caused the entire app to reload while the update request was still in flight
- All in-memory state, including the accessToken stored in Zustand, was destroyed

##### 2. Auth State Corruption During Reload

```typescript
// EditCompanyModal.tsx - PROBLEMATIC CODE
const handleUpdate = async (e: React.FormEvent) => {
  // ...
  await updateCompany(formDataToSend, company._id);
  globalThis.location.reload(); // ❌ Destroys auth state
};
```

When the page reloaded:

- `App.tsx` remounted and called `refreshAuth()`
- The access token was gone from memory
- The refresh token was retrieved from Electron secure storage
- However, due to timing issues during reload, the token was either:
    - Stale/corrupted from a previous session
    - Not fully persisted from a recent refresh
    - Racing with another token update operation

##### 3. Token Verification Failure Chain

```
Time 0ms:   User clicks "Save Changes"
Time 10ms:  Update request sent
Time 20ms:  location.reload() called (doesn't wait for response)
Time 30ms:  Page starts reloading → Zustand state destroyed
Time 50ms:  App.tsx remounts
Time 60ms:  refreshAuth() called (no accessToken in memory)
Time 70ms:  Retrieves refresh token from secure storage
Time 80ms:  Token is invalid/corrupted/stale
Time 90ms:  Backend jwt.verify() fails with "Invalid signature"
Time 100ms: axios interceptor catches 401 → logout()
```

##### 4. Why the New Axios Interceptor Made It Visible

- Before the interceptor improvements, 401 errors might have silently failed
- The new interceptor properly handles 401s by attempting token refresh
- When refresh fails, it correctly logs the user out as a security measure
- This proper error handling exposed the underlying reload timing issue

#### Discovery Process

1. Initially suspected JWT secret mismatch between environments ❌
2. Traced API calls - request structure was correct
3. Added extensive logging to axios interceptor and refresh flow
4. Noticed the error occurred specifically after company updates
5. Discovered `location.reload()` was being called in modal handlers
6. Realized reload was destroying auth state mid-request
7. Found same anti-pattern in multiple components (CompaniesPage, EditCompanyModal, AddCompanyModal)

### How We Fixed It

#### 1. Removed All `location.reload()` Calls

Replaced hard reloads with proper state management using Zustand's built-in reactivity.

#### 2. Updated Store to Return Success/Failure Booleans

#### 3. Updated Modal Handlers to Wait for Completion

#### 4. Fixed Refresh Button in CompaniesPage

```typescript
// CompaniesPage.tsx:
const refresh = async (): Promise<void> => {
  setIsRefreshing(true);
  await getCompanies(); // ✅ Just refetch data, don't reload page
  setIsRefreshing(false);
};
```

### Prevention/Lessons Learned

#### Key Takeaways

> **Never use `location.reload()` in modern React applications**

Why it's problematic:

- Destroys all in-memory state (auth tokens, form data, UI state)
- Creates race conditions with in-flight API requests
- Breaks single-page application architecture
- Provides poor user experience (flash, loss of scroll position)

#### Proper State Management Practices

- Use Zustand/Redux reactivity for data updates
- Store actions should call `get().getCompanies()` to refresh data naturally
- Components automatically re-render when store data changes
- No need for manual page reloads

#### Async Operation Handling

- Always await API calls before taking UI actions
- Return success/failure booleans from store actions
- Close modals/dialogs only AFTER operations complete
- Show loading states during operations

#### Authentication State Protection

- Never destroy auth state unnecessarily
- Keep access tokens in memory (Zustand)
- Keep refresh tokens in secure storage (Electron secure storage)
- Rely on axios interceptors for token refresh, not manual reloads

#### Error Handling Best Practices

- Proper 401 handling reveals underlying issues (which is good!)
- Silent failures hide bugs - always handle errors explicitly
- Log errors comprehensively for debugging
- Toast notifications for user feedback

#### Code Review Checklist

- [ ] No `location.reload()` or `window.location.reload()` calls
- [ ] All async operations are properly awaited
- [ ] Store actions return success/failure status
- [ ] Modal handlers wait for operations before closing
- [ ] Auth state is never destroyed during normal operations
- [ ] Loading states are shown during async operations

### Related Issues

N/A

---

## Error #2 - Docker daemon - Host EC2 kernel conflict

**Date Encountered:** 28th Dec 2025  
**Severity:** 🔴 Critical  
**Environment:** AWS EC2 / Development  
**Status:** ✅ Resolved

### Error Description

While setting up Jenkins on a docker container on an EC2 instance to run Maven + Docker builds, the system became completely unresponsive after attempting to install Docker inside the Jenkins container.

**Symptoms:**

- Jenkins jobs started failing after Docker installation
- Jenkins UI became extremely laggy and unresponsive
- Browser tab forced to close
- EC2 instance became unresponsive via SSH
- Jenkins container would stop/crash repeatedly
- Required closing terminal and re-SSHing to regain access

### Stack/Technology

- **Infrastructure:** AWS EC2 instance
- **CI/CD:** Jenkins running in Docker container
- **Build Tool:** Maven
- **Containerization:** Docker (attempted Docker-in-Docker)

### Root Cause

**Attempted to install Docker daemon inside the Jenkins container, creating a conflicting Docker-in-Docker scenario.**

#### Why this caused the crash:

1. **Docker requires kernel-level access** - Docker daemon needs direct access to cgroups, namespaces, iptables, and the host kernel
2. **Containers don't have their own kernel** - They're just isolated process trees running on the host's kernel
3. **Two Docker daemons conflicted** - The host Docker daemon and the newly installed container Docker daemon fought for control of the same resources
4. **Host-level resource exhaustion** - Docker daemon misbehavior leaked into the host kernel, causing memory spikes, network stack freezes, and system-wide lag

#### The workflow attempted:

```bash
# Mounted Docker socket (correct)
-v /var/run/docker.sock:/var/run/docker.sock

# Then incorrectly installed Docker inside container (wrong)
curl https://get.docker.com/ > dockerinstall
chmod 777 dockerinstall
./dockerinstall  # as root inside container
```

This created two competing Docker daemons trying to manage the same kernel resources.

### How We Fixed It

1. Stopped and removed the Jenkins container to clear the corrupted state
2. Removed Docker daemon from inside the container - never install dockerd inside Jenkins
3. Used the correct architecture:
    - Docker installed and running on the EC2 host
    - Jenkins container only needs Docker CLI (not daemon)
    - Mount Docker socket to allow Jenkins to communicate with host Docker

#### Correct Jenkins container run command:

```bash
docker run -d \
  -p 8080:8080 \
  -p 50000:50000 \
  -v jenkins_home:/var/jenkins_home \
  -v /var/run/docker.sock:/var/run/docker.sock \
  --name jenkins \
  jenkins/jenkins:lts
```

4. Verified Docker CLI was available inside Jenkins (most Jenkins images include it by default)
5. Tested Docker commands in Jenkins jobs - they now work by communicating with the host Docker daemon through the socket

### Prevention/Lessons Learned

> **Key principle:** Never install Docker daemon inside a container for CI/CD purposes. The container only needs docker CLI and the docker socket file, it uses the daemon on the host machine.

#### Correct architecture:

- **Docker daemon** → runs on the host (EC2 instance)
- **Docker CLI** → available inside Jenkins container
- **Docker socket** → mounted to connect CLI to daemon

#### Why this works:

```
Jenkins container → Docker CLI → /var/run/docker.sock → Host Docker daemon → Host kernel
```

#### Security considerations:

- Mounting Docker socket gives Jenkins root-equivalent access to the host
- This is standard practice for CI/CD but understand the security implications
- Never download and run installation scripts with `chmod 777` without reviewing them

### Related Issues

- Docker-in-Docker (DinD) vs Docker-out-of-Docker (DooD) architecture patterns
- Jenkins container security best practices
- EC2 instance kernel resource management

---

## Error #3 - E-Commerce Project Postponed

**Date Encountered:** January 7th 2026  
**Project Duration:** August 2025 - January 2026 (5 months)  
**Reported By:** Rasyar Safin Mustafa  
**Severity:** 🔴 Critical - Project Level  
**Environment:** Full-stack E-commerce Platform

### Error Description

A multi-platform e-commerce project was postponed after 5 months of development due to fundamental business requirements remaining undefined.

**The project consisted of:**

- Express.js backend API
- Flutter mobile app (buyer-facing)
- Desktop application (admin/office management)

**Critical undefined requirements included:**

- Can orders contain products from multiple cities/providers?
- How is product size/weight calculated and stored?
- How is delivery cost calculated? (per kilometer? fixed per city? weight-based?)
- What are the business rules for multi-city fulfillment?

When asked for clarification, the business owner (new to business) could not provide clear answers. The project reached a standstill where the team could not proceed without these fundamental decisions.

#### Team conflict emerged:

- **One view:** "Requirements were too loosely defined from the start - we shouldn't have begun without clarity"
- **Another view:** "We developed too slowly - we should have surfaced these questions months earlier"

### Stack/Technology

- **Backend:** Express.js (Node.js)
- **Mobile:** Flutter
- **Desktop:** Electron/Desktop framework
- **Database:** MongoDB
- **Project timeline:** 5 months with no completion in sight

### Root Cause

**The project started without a proper discovery/requirements phase.**

#### Primary failure points:

1. **No requirements validation** - Development began before core business logic was defined
2. **Insufficient stakeholder engagement** - Business owner was not pushed to clarify their business model before coding started
3. **No gated project phases** - No checkpoint requiring signed-off requirements before development
4. **Late surfacing of complexity** - Order fulfillment, delivery logistics, and multi-city operations were not addressed early

#### Contributing factors:

- Business owner was inexperienced and didn't understand their own business model requirements
- Development team didn't enforce a discovery phase or refuse to start without clarity
- No formal project management or requirements documentation process
- Assumption that requirements would "emerge" during development

#### The "too slow development" argument:

Partially valid if 5 months passed without building the complex order/delivery features that would have surfaced these questions earlier. However, building without clear requirements would have resulted in throwing away significant code regardless of speed.

### How We Fixed It (Or Should Fix It)

#### Immediate actions:

1. Halt all development until requirements are defined
2. Schedule requirements workshops with business owner
3. Document every business rule before resuming development
4. Create a requirements sign-off document

#### Proper project restart process:

##### Phase 1: Discovery (2-4 weeks)

- Conduct stakeholder workshops with business owner
- Map complete user journeys for buyers and admins
- Define ALL business rules:
    - Order fulfillment logic
    - Delivery cost calculation methods
    - Multi-city/provider handling
    - Product catalog structure (weight, size, categories)
    - Payment and refund flows
- Document assumptions and get explicit sign-offs
- Identify what the business owner doesn't know and help them research industry standards

##### Phase 2: Technical Design (1-2 weeks)

- Design database schema based on confirmed requirements
- Define API contracts
- Create architecture diagrams
- Estimate development timeline with known scope

##### Phase 3: Development (Iterative)

- Build in priority order
- Regular demos to validate understanding
- Establish a formal change request process

##### Phase 4: Launch & Iterate

- MVP launch with core features
- Gather real user feedback
- Iterate on feedback, not assumptions

### Prevention/Lessons Learned

> **Critical principle:** Never start development on a project with undefined business requirements, especially for complex domains like e-commerce logistics.

#### Requirements red flags to watch for:

- ⚠️ Business owner can't answer basic "how does X work?" questions
- ⚠️ Stakeholders say "we'll figure it out as we go"
- ⚠️ No written requirements document
- ⚠️ Business owner is new to the industry with no research done
- ⚠️ Vague acceptance criteria like "make it work like Amazon"

#### What we should have done:

1. Conducted a paid discovery phase before committing to development
2. Created a requirements document and required sign-off
3. Built a proof-of-concept for the most complex features first (order + delivery logic) to surface questions immediately
4. Established a formal change request process from day one
5. Educated the business owner on what decisions needed to be made before building

#### Developer responsibility:

- It is **not** "moving fast" to build on unclear requirements
- Developers must push back and demand clarity before coding
- "I don't know yet" from a client means "stop development" not "keep building and hope"
- Surface the hard questions in week 1, not month 7

#### The cost of this error:

- 5 months of partially wasted development effort
- Damaged client relationship
- Team morale impact
- Opportunity cost of other projects

#### Why this happens:

- Eagerness to start coding and show progress
- Fear of appearing difficult by demanding requirements
- Assumption that requirements will become clear through development
- Inexperienced clients who don't know what they don't know
- No formal project management discipline

#### Key quote to remember:

> "Hours spent in requirements save weeks spent in development and months spent in rework."

### Related Issues

- Scope creep and change management
- Client communication and expectation setting
- Agile vs. Waterfall for projects with undefined requirements
- When to walk away from a project

---

**Tags:** #errors #documentation #lessons-learned #best-practices



