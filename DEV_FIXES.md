# Oslira Development Tasks & Issues

## Critical Global Issues

### Performance & Architecture
- **Slow Script Initialization** - Migrate to dependency-readiness pattern to prevent partial loading and styling issues
- **Frontend Refactoring** - Modularize codebase with intelligent file organization for long-term maintainability
- **Backend Refactoring** - Modularize codebase with intelligent file organization for long-term maintainability
- **Type Normalization** - Search through frontend and backend code to normalize types (quick, profile, xray)
- **Plan Terminology** - Remove all instances of "starter plan" and normalize to "free" across frontend and backend

### Infrastructure & Monitoring
- **Staging Protection** - Re-add staging protection to staging website
- **Sentry Integration** - Implement comprehensive error tracking from frontend to backend for admin visibility
- **R2 Bucket Cleanup** - Fix and confirm why R2 bucket isn't clearing leads after 1 day
- **Alert System** - Utilize alert system to keep users informed of process status

### Documentation & Configuration
- **Documentation Repository** - Build out comprehensive docs repo
- **GitHub Actions Guardrails** - Create `.yml` guardrails for all repositories
- **Development Console** - Create all devconsole files with detailed testing coverage for every feature
- **CI/CD Configuration** - Build out all `.yml` workflow files

### User Experience
- **Tutorial System** - Add tutorial and track `tutorial_completed` status for users
- **Favicon** - Add favicon to every page
- **Footer Pages** - Rework CSS for all footer pages
- **Sidebar Navigation** - Change sidebar to use "Integrations" instead of "Automation"
- **URL System** - Fix all hrefs to use new routing system
- **CSS Organization** - Move all CSS from scattered files to `src/components` directory structure
- **Markdown Rendering** - Find solution for proper markdown spacing and color support

---

## Priority Implementation (22 Hours Total)

### Core System Improvements
1. **Vite Build** (8 hrs) - Achieve 90% faster load times
2. **API Versioning** (2 hrs) - Prevent production breaking changes
3. **Schema Validation** (4 hrs) - Catch API bugs early
4. **Cache Invalidation** (3 hrs) - Eliminate stale data issues
5. **Sentry Configuration** (1 hr) - Enhanced error tracking setup
6. **Rate Limiter Client** (4 hrs) - Implement abuse prevention

**Expected Result:** Production-ready bulletproof system in 3 weeks

---

## Home Page Issues

### Bugs
- **Red Line Bug** - Red line appears under limited offer banner when errors occur on home page

### Features
- **Anonymous Analysis** - Implement real anonymous analysis functionality
- **Handler Registration** - Put home handlers in registry and integrate properly
- **Modal Consistency** - Analysis should display the same modal as dashboard

---

## Authentication Issues

### Critical Bugs
- **Login Loop** - Sometimes after login, redirects to dashboard then immediately back to login page
  ```javascript
  // Test command:
  localStorage.clear();
  sessionStorage.clear();
  location.href = '/';
  ```

### Features Needed
- **Persistent Sessions** - Implement continuous sessions that remember login status for up to 3 days

---

## Onboarding Issues

### UI Improvements
- **Progress Indicator** - Copy queue implementation of percentage complete and estimated loading time (once fixed)
- **Error Handling** - Display errors when fields are not selected and user clicks "Next" instead of silent failure

---

## Dashboard Issues

### Critical Bugs
- **Tab Stability** - Switching/refreshing/closing/any tab alteration stops bulk and single analysis - make processes resilient to tab changes

### Bulk Analysis
- **Functionality Testing** - Test why bulk analysis isn't working
- **Proper Implementation** - Implement bulk analysis with usage tracking
- **Manual Upload** - Add manual upload capability to bulk analysis
- **Tutorial Animation** - Add animated tutorial showing how bulk analysis works

### Analysis Features
- **Personality Section** - Add personality section at the top of analysis
- **Enhanced Personality** - Allocate more tokens and make personality generation significantly more detailed
- **Profile Picture** - Raise profile picture positioning inside analysis modal
- **Comments Tab** - Add comments tab next to personality section or integrate into CRM

### Analysis Queue
- **Queue Persistence** - Analysis queue should retain leads and process them through page refreshes
- **Stop Controls** - Add "Stop All" button and individual stop controls for each lead
- **UI Controls** - Fix little chevron and X button functionality on analysis queue
- **Progress Flow** - Fix percentage calculation and display flow
- **Conditional Display** - Analysis queue should be hidden unless items are queued

### UI & Filters
- **Stat Card Enhancement** - Add stat card for promoting leads from lead research page to main table
- **Badge System** - Add badges integrated with saved lists card (rename and update functionality)
- **Badge Sources** - Pull from business niche/Apify fields or use AI generation for visually appealing badges
- **Additional Filter** - Add filter between "All Platforms" and "Recent" to sort by analysis type

---

## Settings Page Issues

### Implementation
- **Page Development** - Implement complete settings page
- **Credit Logs** - Allow users to download credit logs or display recent 10 transactions with download option
- **Info Migration** - Steal relevant information from subscription page and disperse throughout settings

---

## Subscription Page Issues

### Restructuring
- **Simplified Upgrade** - Use simple `/upgrade` page instead of complex subscription flow
- **Upgrade Functionality** - Implement upgrade page functionality
- **Bulk Analysis Limits** - Make bulk analysis limits a subscription tier feature
- **Limit Detection** - Implement detection with user-friendly messaging: "Too many usernames, your plan allows X"
- **Info Migration** - Migrate all subscription info to settings page to clean up pricing page

---

## Footer: Contact Page Issues

*(To be defined)*

---

## Footer: Documentation Issues

### Implementation
- **Build from Repo** - Build out after `/docs` repo is complete and utilize that information

---

## Footer: Help Page Issues

*(To be defined)*

---

## Footer: Legal Page Issues

*(To be defined)*

---

## Footer: Other Issues

### Pricing & Plans
- **Trial Offer** - Consider $1 trial for one month, send as special offers from pricing page
- **Enterprise Pricing** - Change enterprise tier to display price instead of "Contact Sales"
- **Pricing Copy** - Update all pricing page text for clarity

### CTA Buttons
- **Free Trial Button** - Change "Start Free Trial" to "Start Now Free" and redirect to `/auth`
- **Demo Button** - "Schedule Demo" should redirect to `/contact`

### Content
- **Case Studies** - Set up fake data in case studies section
- **Remove Docs Request** - Remove "Request Docs" option
- **Security Contact** - "Contact Security Team" button redirects to `/contact/security`

### Status Page
- **Uptime Display** - When styling reworked, ensure it displays real Uptime Robot information
- **Latency Metrics** - Include latency metrics if implementing enhanced status page

---

## Worker Issues

### Error Handling
- **Profile Not Found** - Fix error handling to send custom status code (not 404 or 200) that informs frontend properly
- **Token Charging** - Only charge 1 token for profile not found errors

### Cost & Performance Tracking
- **Comprehensive Costs** - Track costs from all sources including Apify (even as static cost)
- **Time Tracking** - Implement accurate time tracking across all operations

---

## Admin Panel Issues

### Integration
- **Sentry Integration** - Integrate Sentry for error monitoring in admin panel
- **Uptime Robot** - Display Uptime Robot metrics in admin dashboard

### Metrics
- **Accurate Costs** - Display more accurate cost breakdown including Apify and other third-party services

---

## Future Frontend Implementation

### Lead Management
- **History Tracking** - Implement history tracking on leads for re-research
  - Store raw runs without comparison
  - Add optional "Compare to Previous" button in UI
  - Run comparison analysis on-demand when user clicks

### New Pages
- **Lead Research Page** - Dedicated page for lead research workflow
- **Lead Analytics Page** - Comprehensive analytics dashboard for lead insights

---

## Future Backend Implementation

*(To be defined)*

---

## Notes

This document represents the comprehensive development roadmap for Oslira. Tasks are organized by priority and functional area. Critical items are marked and should be addressed first, followed by the 22-hour priority implementation plan.

**Last Updated:** October 28, 2025
