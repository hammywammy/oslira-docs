🔐 AWS SECRETS MANAGER - OSLIRA CONFIGURATION
Status: ✅ Production Ready
Last Updated: 2025-01-20
Region: us-east-1 (US East - N. Virginia)

📋 NAMING CONVENTION
Pattern:
Oslira/{environment}/{SECRET_NAME}
Environments:

production - Live production environment
staging - Testing/staging environment


🗂️ COMPLETE SECRETS INVENTORY
✅ PRODUCTION SECRETS (14 total)
Secret NamePurposeStatusOslira/production/SUPABASE_URLSupabase project URL✅ RequiredOslira/production/SUPABASE_ANON_KEYSupabase anon key (RLS enforced)✅ RequiredOslira/production/SUPABASE_SERVICE_ROLE_KEYSupabase service role (bypasses RLS)✅ RequiredOslira/production/STRIPE_SECRET_KEYStripe API secret key✅ RequiredOslira/production/STRIPE_PUBLISHABLE_KEYStripe publishable key⚠️ OptionalOslira/production/STRIPE_WEBHOOK_SECRETStripe webhook signature✅ RequiredOslira/production/OPENAI_API_KEYOpenAI API key✅ RequiredOslira/production/ANTHROPIC_API_KEYAnthropic Claude API key✅ RequiredOslira/production/APIFY_API_TOKENApify scraping API✅ RequiredOslira/production/SENTRY_DSNSentry error tracking⚠️ OptionalOslira/production/FRONTEND_URLFrontend application URL⚠️ OptionalOslira/production/ADMIN_TOKENAdmin dashboard auth⚠️ Optional

✅ STAGING SECRETS (12 total)
Secret NamePurposeOslira/staging/SUPABASE_URLStaging Supabase projectOslira/staging/SUPABASE_ANON_KEYStaging anon keyOslira/staging/SUPABASE_SERVICE_ROLE_KEYStaging service roleOslira/staging/STRIPE_SECRET_KEYStripe test mode keyOslira/staging/STRIPE_PUBLISHABLE_KEYStripe test publishableOslira/staging/STRIPE_WEBHOOK_SECRETStripe test webhookOslira/staging/OPENAI_API_KEYOpenAI keyOslira/staging/ANTHROPIC_API_KEYAnthropic Claude keyOslira/staging/APIFY_API_TOKENApify tokenOslira/staging/SENTRY_DSNStaging SentryOslira/staging/FRONTEND_URLStaging frontend URLOslira/staging/ADMIN_TOKENStaging admin token

📊 SECRETS USAGE BY WORKER
Phase 0.2 (Current):
typescriptawait getSecret('SUPABASE_URL', env, env.APP_ENV);
await getSecret('SUPABASE_ANON_KEY', env, env.APP_ENV);
await getSecret('SUPABASE_SERVICE_ROLE_KEY', env, env.APP_ENV);
Phase 1+ (Future):
typescript// AI providers
await getSecret('OPENAI_API_KEY', env, env.APP_ENV);
await getSecret('ANTHROPIC_API_KEY', env, env.APP_ENV);

// Scraping
await getSecret('APIFY_API_TOKEN', env, env.APP_ENV);

// Payments
await getSecret('STRIPE_SECRET_KEY', env, env.APP_ENV);
await getSecret('STRIPE_WEBHOOK_SECRET', env, env.APP_ENV);

// Monitoring (optional)
await getSecret('SENTRY_DSN', env, env.APP_ENV);

🔐 SECURITY BEST PRACTICES
✅ DO:

Keep all API keys in AWS Secrets Manager
Use separate keys for staging vs production
Rotate secrets quarterly
Enable CloudTrail logging

❌ DON'T:

Never commit secrets to git
Never hardcode secrets in code
Never log secret values
Never share production keys


🧪 VERIFICATION COMMANDS
List all Oslira secrets:
bashaws secretsmanager list-secrets --region us-east-1 \
  --query 'SecretList[?starts_with(Name, `Oslira/`)].Name' \
  --output table
Verify specific secret:
bashaws secretsmanager describe-secret \
  --secret-id Oslira/production/SUPABASE_SERVICE_ROLE_KEY \
  --region us-east-1

📝 NOTES

Region: All secrets in us-east-1
Cache TTL: 5 minutes per Worker instance
Cost: ~$0.40/month per secret
Total Cost: ~$10.40/month (26 secrets)
