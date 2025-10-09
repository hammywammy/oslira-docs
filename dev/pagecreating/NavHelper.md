NAVIGATION HELPER - COMPLETE USAGE GUIDE

WHAT IT DOES

✅ Auto-detects environment (production/staging) and builds correct URLs
✅ Initializes all data-nav links automatically on page load
✅ Watches for dynamic content and auto-updates new links
✅ Provides programmatic navigation API for JavaScript
✅ Tracks navigation events for analytics
✅ Handles query parameters and hash fragments
✅ Detects external links and adds security attributes
✅ Built-in auth checking for protected routes


INSTALLATION
Step 1: Add to your loader
File: public/core/init/ModuleRegistry.js
Add this to your core scripts array:
javascript'/core/utils/NavigationHelper.js'
Step 2: Load order
NavigationHelper must load AFTER EnvDetector:
javascript// Load order in your loader:
1. EnvDetector.js
2. NavigationHelper.js  ← Add here
3. Other modules...

HTML USAGE
Basic Navigation Links
html<!-- Auth/Login Links -->
<a href="#" data-nav="auth">Login</a>
<a href="#" data-nav="signin">Sign In</a>
<a href="#" data-nav="signup">Sign Up</a>

<!-- App Links -->
<a href="#" data-nav="dashboard">Dashboard</a>
<a href="#" data-nav="leads">Leads</a>
<a href="#" data-nav="analytics">Analytics</a>
<a href="#" data-nav="campaigns">Campaigns</a>
<a href="#" data-nav="messages">Messages</a>
<a href="#" data-nav="settings">Settings</a>

<!-- Marketing Links -->
<a href="#" data-nav="home">Home</a>
<a href="#" data-nav="about">About</a>
<a href="#" data-nav="pricing">Pricing</a>
<a href="#" data-nav="help">Help</a>

<!-- Legal Links -->
<a href="#" data-nav="terms">Terms of Service</a>
<a href="#" data-nav="privacy">Privacy Policy</a>
<a href="#" data-nav="refund">Refund Policy</a>

<!-- Contact Links -->
<a href="#" data-nav="contact">Contact</a>
<a href="#" data-nav="contact-support">Support</a>
<a href="#" data-nav="contact-sales">Sales</a>
<a href="#" data-nav="contact-bug-report">Report a Bug</a>

<!-- Status -->
<a href="#" data-nav="status">System Status</a>
What happens:

Production: <a href="https://auth.oslira.com">Login</a>
Staging: <a href="https://staging-auth.oslira.com">Login</a>


Custom Paths
html<!-- Custom path within target -->
<a href="#" data-nav="app" data-nav-path="/subscription">Upgrade</a>
<a href="#" data-nav="legal" data-nav-path="/disclaimer">Disclaimer</a>
<a href="#" data-nav="settings" data-nav-path="/billing">Billing Settings</a>

Advanced Options
html<!-- Preserve query parameters -->
<a href="#" data-nav="dashboard" data-nav-query="true">Dashboard</a>

<!-- Don't preserve query parameters -->
<a href="#" data-nav="auth" data-nav-query="false">Clean Login</a>

<!-- Preserve hash fragments -->
<a href="#" data-nav="help" data-nav-hash="true">Help (same section)</a>

<!-- Intercept clicks (for custom handling) -->
<a href="#" data-nav="dashboard" data-nav-intercept>Dashboard</a>

Buttons
html<!-- Button with onclick -->
<button onclick="window.OsliraNav.navigateTo('auth')">
    Login
</button>

<button onclick="window.OsliraNav.navigateTo('dashboard')">
    Go to Dashboard
</button>

<button onclick="window.OsliraNav.navigateTo('settings', '/billing')">
    Manage Billing
</button>

JAVASCRIPT USAGE
Basic Navigation
javascript// Navigate to auth
window.OsliraNav.navigateTo('auth');

// Navigate to dashboard
window.OsliraNav.navigateTo('dashboard');

// Navigate to specific settings page
window.OsliraNav.navigateTo('settings', '/billing');

// Navigate to legal page
window.OsliraNav.navigateTo('terms');

Get URLs Without Navigating
javascript// Get single URL
const authUrl = window.OsliraNav.getUrl('auth');
console.log(authUrl); // "https://staging-auth.oslira.com" (in staging)

const dashboardUrl = window.OsliraNav.getUrl('dashboard');
console.log(dashboardUrl); // "https://staging-app.oslira.com/dashboard" (in staging)

// Get custom path
const billingUrl = window.OsliraNav.getUrl('settings', '/billing');
console.log(billingUrl); // "https://staging-app.oslira.com/settings/billing"

// Get multiple URLs at once
const urls = window.OsliraNav.getUrls(['auth', 'dashboard', 'terms']);
console.log(urls);
// {
//   auth: "https://staging-auth.oslira.com",
//   dashboard: "https://staging-app.oslira.com/dashboard",
//   terms: "https://staging-legal.oslira.com/terms"
// }

Navigation with Options
javascript// Open in new tab
window.OsliraNav.navigateTo('help', '', { newTab: true });

// Preserve query parameters
window.OsliraNav.navigateTo('dashboard', '', { preserveQuery: true });

// Preserve hash fragment
window.OsliraNav.navigateTo('help', '', { preserveHash: true });

// Combine options
window.OsliraNav.navigateTo('pricing', '', { 
    newTab: true, 
    preserveQuery: true 
});

Navigation with Return URL
javascript// Navigate to auth with return URL (for redirecting back after login)
window.OsliraNav.navigateWithReturn('auth');

// This creates: https://staging-auth.oslira.com?return_to=<current_url>

Require Authentication
javascript// Check if user is authenticated, redirect to auth if not
if (!window.OsliraNav.requireAuth()) {
    // User is not authenticated, will be redirected to auth
    console.log('Redirecting to auth...');
}

// Require auth with custom redirect
window.OsliraNav.requireAuth('https://staging-app.oslira.com/dashboard');

Utility Methods
javascript// Get all available navigation targets
const targets = window.OsliraNav.getAvailableTargets();
console.log(targets);
// ['auth', 'login', 'dashboard', 'leads', 'settings', 'terms', ...]

// Check if target exists
if (window.OsliraNav.hasTarget('dashboard')) {
    console.log('Dashboard target available');
}

// Get target configuration
const config = window.OsliraNav.getTargetConfig('dashboard');
console.log(config);
// { method: 'getAppUrl', path: '/dashboard', requiresAuth: true }

// Refresh all navigation links on page (useful after dynamic content loaded)
window.OsliraNav.refreshAll();

// Debug information
window.OsliraNav.debug();
