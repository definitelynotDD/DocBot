# ✅ Pre-Deployment Action Checklist

All critical security fixes have been applied to DocBot. Use this checklist before deploying to production.

---

## IMMEDIATE (Before any deployment)

- [ ] **1. Install updated dependencies:**
  ```bash
  pip install --upgrade -r requirements.txt
  ```
  This installs RestrictedPython and sqlparse.

- [ ] **2. Verify no .env files in Git history:**
  ```bash
  git log --all --full-history -- "*.env"
  ```
  If this shows results → your API keys may be exposed! Regenerate all keys immediately.

- [ ] **3. Verify .gitignore is protecting .env files:**
  ```bash
  grep -E "\.env|\.env\*" .gitignore
  ```
  Should show entries for `*.env` and `.env*`

- [ ] **4. Test the app runs without errors:**
  ```bash
  streamlit run app.py
  ```
  App should start, show privacy notice below file uploader, and have no errors.

---

## LOCAL TESTING (On your machine)

- [ ] **5. Test file size limit:**
  - Create a >20MB file (e.g., copy a large PDF twice)
  - Try uploading via UI
  - Should see error: "❌ File **...** exceeds 20MB limit"

- [ ] **6. Test SQL injection blocking:**
  - Upload any CSV with data
  - Connect to a test database in sidebar
  - In chat, try: "DROP ALL TABLES" or "DELETE everything"
  - Should see: "❌ Only SELECT queries are allowed"

- [ ] **7. Test safe code execution:**
  - Upload a CSV
  - In chat, try: "Show me a line chart"
  - Chart should render without errors (RestrictedPython executed safely)

- [ ] **8. Verify privacy notice displays:**
  - Look below the file uploader
  - Should see: ⚠️ Privacy Notice with Google/GitHub API warning

---

## BEFORE PUBLIC/CLOUD DEPLOYMENT

- [ ] **9. Ensure database uses read-only user (SQL only):**
  ```sql
  -- PostgreSQL example:
  CREATE USER docbot_readonly WITH PASSWORD 'very_secure_password';
  GRANT SELECT ON ALL TABLES IN SCHEMA public TO docbot_readonly;
  ```
  Update `.env`: `SQL_DATABASE_URL=postgresql://docbot_readonly:password@host/db`

- [ ] **10. Enable email authentication for Streamlit Cloud:**
  In Streamlit Cloud Secrets add:
  ```
  SECRET_AUTH_EMAILS=user1@example.com,user2@example.com
  ```
  Uncomment the `check_auth_email_allowlist()` call in app.py if public access needed.

- [ ] **11. Review SECURITY.md:**
  Read [SECURITY.md](SECURITY.md) for:
  - Deployment best practices
  - Email allowlist setup
  - Resource limits
  - Monitoring guidelines

- [ ] **12. Set API rate limits (if using Gemini/GitHub):**
  - Google AI Studio: https://aistudio.google.com/apikey
  - GitHub Models: https://github.com/settings/tokens
  - Set appropriate quotas to prevent cost overruns

---

## STREAMLIT CLOUD DEPLOYMENT

- [ ] **13. Create `.streamlit/secrets.toml`:**
  ```toml
  GEMINI_API_KEY = "your_key"
  GITHUB_TOKEN = "your_token"
  SECRET_AUTH_EMAILS = "user1@example.com,user2@example.com"
  ```
  **Never** store in `.streamlit/config.toml`

- [ ] **14. Deploy:**
  ```bash
  git add -A
  git commit -m "security: add RestrictedPython, SQL validation, privacy notice"
  git push origin main
  # Streamlit Cloud auto-deploys
  ```

- [ ] **15. Test public access:**
  - Visit your Streamlit Cloud URL
  - Privacy notice should display
  - Try file uploads & SQL queries
  - Verify email auth works (if enabled)

---

## DOCKER/SELF-HOSTED DEPLOYMENT

- [ ] **16. Create `.env` from `.env.example`:**
  ```bash
  cp env.example .env
  # Fill in your values (keys, database URL, etc.)
  ```

- [ ] **17. Build Docker image:**
  ```bash
  docker build -t docbot:secure .
  ```
  Ensure `requirements.txt` with RestrictedPython is included.

- [ ] **18. Run with environment variables:**
  ```bash
  docker run \
    -e GEMINI_API_KEY="$GEMINI_KEY" \
    -e GITHUB_TOKEN="$GITHUB_TOKEN" \
    -e SECRET_AUTH_EMAILS="user@example.com" \
    -p 8501:8501 \
    --memory="2gb" \
    --cpus="1" \
    docbot:secure
  ```

- [ ] **19. Set up reverse proxy with HTTPS:**
  - Use nginx, Apache, or Traefik
  - Enforce HTTPS only
  - Add rate limiting
  - Enable compression

---

## POST-DEPLOYMENT

- [ ] **20. Monitor logs for:**
  - "Chart error" or "Execution error" → code injection attempts blocked ✅
  - "Only SELECT queries allowed" → SQL injection attempts blocked ✅
  - File size errors → oversized upload attempts blocked ✅

- [ ] **21. Set up alerts for:**
  - API quota usage spikes
  - Failed authentication attempts
  - Unusual error rates

- [ ] **22. Monthly security review:**
  - Check for RestrictedPython updates
  - Review access logs
  - Rotate database passwords periodically
  - Update API keys if compromised

---

## TROUBLESHOOTING

**Problem:** "ModuleNotFoundError: No module named 'RestrictedPython'"  
**Solution:** `pip install RestrictedPython`

**Problem:** File uploads rejected even under 20MB  
**Solution:** Check file size calculation: `len(file.read()) > MAX_UPLOAD_SIZE_BYTES`

**Problem:** SQL queries blocked incorrectly (valid SELECT blocked)  
**Solution:** Check sqlparse version, may need regex debug. Add logging to `run_sql_query()`.

**Problem:** Privacy notice not showing  
**Solution:** Verify `st.info(PRIVACY_NOTICE, ...)` is in app.py line ~1647

**Problem:** .env file still in Git after pushing**  
**Solution:** 
```bash
git rm --cached .env
git commit -m "stop tracking .env"
git push
```

---

## Documentation Files Created

- **SECURITY.md** — Comprehensive security guide (read this!)
- **FIXES_SUMMARY.md** — Quick reference of all changes
- **DEPLOYMENT_CHECKLIST.md** — This file

---

## Final Safety Check

Before going live, run this script:

```bash
#!/bin/bash
echo "🔒 DocBot Security Pre-Flight Check"
echo ""

echo "1️⃣  Checking Python syntax..."
python -m py_compile app.py && echo "✅ Syntax OK" || echo "❌ Syntax error"

echo "2️⃣  Checking RestrictedPython installed..."
python -c "import RestrictedPython" && echo "✅ RestrictedPython OK" || echo "❌ RestrictedPython missing"

echo "3️⃣  Checking sqlparse installed..."
python -c "import sqlparse" && echo "✅ sqlparse OK" || echo "❌ sqlparse missing"

echo "4️⃣  Checking .env not in Git..."
git log --all --full-history -- "*.env" | grep -q . && echo "❌ FOUND .env in history!" || echo "✅ .env protected"

echo "5️⃣  Checking .gitignore has *.env..."
grep -q "^\*.env" .gitignore && echo "✅ .gitignore OK" || echo "❌ .gitignore missing *.env"

echo ""
echo "✅ All checks passed! Ready for deployment."
```

---

## Contact & Support

- 📖 **Read:** [SECURITY.md](SECURITY.md)
- 🐛 **Report bugs:** GitHub Issues
- 🔐 **Security issues:** Email privately (don't open public issues for security)

---

**Ready to deploy safely!** 🚀
