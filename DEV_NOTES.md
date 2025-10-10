🔴 GLOABAL ISSUES #GLOBAL
-slow script initialization either leading to things not working or displaying or displaying with proper styling until everything is loaded (migrate to dependency-readiness)
-refactor front end code, modularize and cleanly and intelligently store files thorughout, build it out this time for the long run
-refactor back end code, modularize and cleanly and intelligently store files thorughout, build it out this time for the long run
-search through front and back end code and normalize types quick, profile, xray
-search through fron tand back end code and remove any instances of starter plan, normalize free
-add staging protection back to staging website
-integrate sentry to catch all errors from frontend to backend, so that admin can have access to real issues
-fix and confirm why r2 bucket not clearing leads after 1 day
-utilize alert_system for users to know whats happening
-build out docs repo
-build out .yml guardrails for the repos
-add tutorial and users tutorial_completed 
-add favicon to every page
-rework all footer pages css
-change sidebar to use integrations not automation
-fix all hrefs to use new system
-create all devconsole files and get detailed with every nook and cranny for testing
-build out all .yml files
-move all css from all files to src/components, whether thats splitting it up into multiple files or whatever

🟠 HOME ISSUES #HOME
-when the home page has an error a red line appears under the limited offer banner
-home page needs to use real anonymous analysis, put homehandlers in registry and integrate
-analysis should show the same modal as dashboard


🟡 AUTH ISSUES #AUTH
-sometimes ill log in it will go to dashboard and send me right back to the log in page again 
use this cmd to test: 
localStorage.clear();
sessionStorage.clear();
location.href = '/';

-have continuos sessions where it remembers im logged in for up to 3 days
-

 
🟢 ONBOARDING ISSUES #ONBOARDING
-copy my queues implementation of percentage complete and estimated loading time (once its fixed)
-fix to show errors when things are not selected and i click next instead of nothing 
-


🔵 DASHBOARD ISSUES #DASHBOARD
-switching/refreshing/closing/any altering of tabs stops my bulk and single anlaysis, make it actually running

-test why bulk analysis isnt working
-implement bulk analysis proprely with usage tracking
-add manual upload to bulk analysis
-add little anim tutorial showing how it works

-add in personality section at the top
-give more tokens and make personality generation a lot more detailed
-raise profile pic inside analysis modal
-add a comments tab right next to personality or on crm

-analysis queue should actually hold onto leads and process them not auto quit on page refresh
-analysis queue needs a stop all button and a individual stop lead
-get little chevron and x on analysis queue to work 
-fix percentage flow

-add badges, integrate with saved lists card (change name and func), pull from business niche or whatever apify field, or have ai generate (visually appealing)

🟣 SETTINGS PAGE ISSUES #SETTINGS
-implement
-allow users to download credit logs, or display it for recent 10 transaction and sitll have a download button 
-steal info and disperse from subscription page

🟣 SUBSCRIPTION PAGE ISSUES #SUBSCRIPTION
-use a simple /upgrade page instead
-implement upgrade page functionality
-make bulk anlysis limits a feature depending on subscriptin and implement the detection and: too many usernames, your plan allows x 
-all info will be migrated throughout settings and to cleanup the pricing page


🟤 FOOTER CONTACT ISSUES #CONTACT
-

🟤 FOOTER DOCS ISSUES #DOCS
-build out after i have my /docs repo, utilize that info

⚫ FOOTER HELP ISSUES #HELP
-


⚪ FOOTER LEGAL ISSUES #LEGAL
-


⚪ FOOTER OTHER ISSUES #OTHER
-consider $1 trial for a month and those cards could be sent as special offers and from pricing
-enterprise will be a price not contact sales
-update pricing page text

-start free trial button should say start now free and take me to /auth
-schedule demo should send me to /contact
-setup fake data in case-studies 

-remove option request docs
-contact security team button redirects to contact/security

-when styling reworked ensure its displaying the real uptime robot information and also include latency if bougee

🟤 WORKER ISSUES #WORKER
-fix error handling on profile not found, dont send a 404 or a 200 send a custom one that informs the front end that i can use, and only charge 1 token
-cost should use from all including apify, even if thats a static cost, and the same goes for time, find a way to track it
-


🟤 ADMIN ISSUES #ADMIN
-integrate sentry and uptime robot into admin
-display more accurate costs (apify etc)
-


💡 FUTURE IMPLEMENTATION FRONTEND #FUTUREFRONT
-history tracking on leads on re-research,  Store raw runs without comparison, Add optional "Compare to Previous" button in UI, Runs comparison analysis on-demand when user clicks
-lead research page
-lead analytics page
-


💡 FUTURE IMPLEMENTATION WORKER #FUTUREBACK
-
