🔴 GLOABAL ISSUES #GLOBAL
-slow script initialization either leading to things not working or displaying or displaying with proper styling until everything is loaded (migrate to dependency-readiness)
-
-
-
-
-
-
-
-
-

🟠 HOME ISSUES #HOME
-when the home page has an error a red line appears under the limited offer banner
-
-
-
-
-
-
-
-
-

🟡 AUTH ISSUES #AUTH
-sometimes ill log in it will go to dashboard and send me right back to the log in page again 
use this cmd to test: // Sign out and clear all auth data
if (window.OsliraAuth && window.OsliraAuth.handleSignOut) {
    await window.OsliraAuth.handleSignOut();
} else if (window.OsliraAuth?.supabaseClient) {
    await window.OsliraAuth.supabaseClient.auth.signOut();
}
// Clear all storage
localStorage.clear();
sessionStorage.clear();
// Reload page
location.href = '/';
-have continuos sessions where it remembers im logged in for up to 3 days
-
-
-
-
-
-
-
-
 
🟢 ONBOARDING ISSUES #ONBOARDING
-copy my queues implementation of percentage complete and estimated loading time (once its fixed)
-
-
-
-
-
-
-
-
-

🔵 DASHBOARD ISSUES #DASHBOARD
-
-
-
-
-
-
-
-
-
-

🟣 SETTINGS PAGE ISSUES #SETTINGS
-
-
-
-
-
-
-
-
-
-

🟤 FOOTER CONTACT ISSUES #CONTACT
-
-
-
-
-
-
-
-
-
-

⚫ FOOTER HELP ISSUES #HELP
-
-
-
-
-
-
-
-
-
-

⚪ FOOTER LEGAL ISSUES #LEGAL
-
-
-
-
-
-
-
-
-
-

🟤 WORKER ISSUES #WORKER
-
-
-
-
-
-
-
-
-
-

💡 FUTURE IMPLEMENTATION FRONTEND #FUTUREFRONT
-
-
-
-
-
-
-
-
-
-

💡 FUTURE IMPLEMENTATION WORKER #FUTUREBACK
-
-
-
-
-
-
-
-
-
-

