# 🔒 Security Hardening Guide for DocBot

This document outlines all security improvements made to DocBot and how to deploy the app safely.

---

## Critical Security Fixes ✅

### 1. Safe Code Execution (Python)

**Problem:** The old `exec(code, {}, ...)` and `exec(code, {"__builtins__": {}}, ...)` calls could be escaped using Python's object model (e.g., `().__class__.__bases__[0].__subclasses__()[104].__init__.__globals__['sys']`).

**Solution:** Now using **RestrictedPython** library for safe code compilation and execution.

- **GRAPH pipeline** (chart generation): Uses `RestrictedPython.compile_restricted()` 
- **DATA pipeline** (pandas operations): Uses `RestrictedPython.compile_restricted()`
- **Fallback**: If RestrictedPython not installed, still uses restricted builtins

**To verify:** Check that `RestrictedPython>=6.0` is in `requirements.txt` and `import RestrictedPython` succeeds.

```python
from restricted_execution import execute_user_code
success, err = execute_user_code(code, {"__builtins__": {}}, local_vars)
```

---

### 2. SQL Injection Prevention

**Problem:** The NL→SQL pipeline had no write-operation protection. A user could prompt the LLM to generate `DROP TABLE` or `DELETE FROM` queries.

**Solution:** Added multiple layers of SQL validation:

1. **sqlparse library** parses the SQL AST and checks `stmt_type == "SELECT"` only
2. **Regex fallback** checks for dangerous keywords: `DROP`, `DELETE`, `INSERT`, `UPDATE`, `CREATE`, `ALTER`, `TRUNCATE`, `EXEC`
3. **Connection-level**: Use a **read-only database user** in production

**Best Practice for Production:**
```sql
-- PostgreSQL example
CREATE USER docbot_readonly WITH PASSWORD 'secure_password';
GRANT CONNECT ON DATABASE mydb TO docbot_readonly;
GRANT USAGE ON SCHEMA public TO docbot_readonly;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO docbot_readonly;

-- Connection URL in .env:
-- SQL_DATABASE_URL=postgresql://docbot_readonly:secure_password@host:5432/mydb
```

**For MySQL:**
```sql
CREATE USER 'docbot_readonly'@'localhost' IDENTIFIED BY 'secure_password';
GRANT SELECT ON mydb.* TO 'docbot_readonly'@'localhost';
```

---

### 3. XSS Prevention (HTML Output)

**Problem:** LLM-generated content could include HTML/JavaScript that gets rendered with `st.markdown(..., unsafe_allow_html=True)`.

**Solution:** 
- Added `escape_llm_content()` helper using `html.escape()`
- Most LLM output is rendered safely (without `unsafe_allow_html=True`)
- Metadata HTML (method, R², etc.) is app-generated and safe

**When to use escaping:**
```python
# UNSAFE ❌
st.markdown(f"<b>{llm_response}</b>", unsafe_allow_html=True)

# SAFE ✅
escaped = escape_llm_content(llm_response)
st.markdown(f"<b>{escaped}</b>", unsafe_allow_html=True)
```

---

### 4. File Upload Size Limits

**Problem:** Large file uploads could cause OOM (out-of-memory) crashes.

**Solution:** Added file size validation in all parsers:

```python
MAX_UPLOAD_SIZE_MB = 20  # 20 MB limit
MAX_UPLOAD_SIZE_BYTES = MAX_UPLOAD_SIZE_MB * 1024 * 1024

def _check_file_size(file_bytes: bytes, filename: str) -> bool:
    """Validate file size. Return True if valid, False if oversized."""
    if len(file_bytes) > MAX_UPLOAD_SIZE_BYTES:
        st.error(f"❌ File exceeds {MAX_UPLOAD_SIZE_MB}MB limit")
        return False
    return True
```

Applied to: `parse_pdf()`, `parse_docx()`, `parse_txt()`, `parse_csv()`, `parse_excel()`

**To adjust limit:** Edit `MAX_UPLOAD_SIZE_MB` in `app.py`

---

### 5. Privacy Notice for API Data Transmission

**Problem:** Users didn't know their files were being sent to Google/GitHub APIs.

**Solution:** Added prominent privacy notice displayed below the file uploader:

```
⚠️ Privacy Notice: Your uploaded files are sent to Google (Gemini) 
and/or GitHub (GPT) APIs for processing. Do not upload sensitive 
personal information, trade secrets, or regulated data (PII, HIPAA, PCI).
```

**Location:** `app.py`, line ~1647

**To customize:** Edit the `PRIVACY_NOTICE` constant in `app.py` lines 119-123

---

### 6. Environment File Protection

**Problem:** `.env` files with API keys could be accidentally committed to Git.

**Solution:** Updated `.gitignore` to block all `.env*` files:

```
*.env
.env*
!.env.example
!env.example
```

**To verify no keys were ever committed:**
```bash
cd c:\Users\HP\Desktop\DocBot
git log --all --full-history -- "*.env" | head -20
# Should show no results
```

**To clean history if keys were committed:**
```bash
git filter-branch --tree-filter 'rm -f .env .env.local' HEAD
git push origin --force --all
# Then regenerate API keys!
```

---

## Deployment Best Practices

### For Streamlit Cloud (Public Deployment)

1. **Enable Email Allowlist Authentication:**
   ```bash
   # In .env or Streamlit Cloud secrets:
   SECRET_AUTH_EMAILS=user1@example.com,user2@example.com
   ```

   The app will reject access from unauthorized emails. (Helper function: `check_auth_email_allowlist()`)

2. **Use Read-Only Database Credentials:**
   ```bash
   # Create a restricted database user (see SQL section above)
   SQL_DATABASE_URL=postgresql://docbot_readonly:password@host/db
   ```

3. **Set Resource Limits:**
   - Streamlit Cloud default: 1GB RAM (should be fine)
   - File upload limit: 20MB (configurable via `MAX_UPLOAD_SIZE_MB`)

4. **Monitor API Usage:**
   - Set Gemini/GitHub API quotas to prevent runaway costs
   - Consider rate limiting per user/session

---

### For Self-Hosted (Docker/VM)

1. **Run Behind Reverse Proxy with Auth:**
   ```nginx
   # nginx with basic auth
   location / {
       auth_basic "Restricted";
       auth_basic_user_file /etc/nginx/.htpasswd;
       proxy_pass http://localhost:8501;
   }
   ```

2. **Use Environment Variables for Secrets:**
   ```bash
   docker run -e GEMINI_API_KEY=$KEY -e GITHUB_TOKEN=$TOKEN docbot:latest
   ```

   **Never** pass secrets as command-line arguments.

3. **Enable HTTPS:**
   ```bash
   streamlit run app.py --client.baseUrl=https://yourdomain.com --logger.level=info
   ```

4. **Set Resource Limits:**
   ```bash
   docker run -m 2gb --cpus="1" docbot:latest
   ```

---

### For Local Development

1. **Copy `.env.example` to `.env`:**
   ```bash
   cp env.example .env
   ```

2. **Add your API keys:**
   ```
   GEMINI_API_KEY=your_key_here
   GITHUB_TOKEN=your_token_here
   ```

3. **Install dependencies:**
   ```bash
   pip install -r requirements.txt
   ```

4. **Run locally:**
   ```bash
   streamlit run app.py
   ```

---

## Verification Checklist

- [ ] `RestrictedPython>=6.0` installed (`pip install -r requirements.txt`)
- [ ] `sqlparse>=0.5.0` installed
- [ ] `.gitignore` includes `*.env` and `.env*`
- [ ] No `.env` files in Git history: `git log --all --full-history -- "*.env"`
- [ ] File size limit tested: Try uploading >20MB file (should error)
- [ ] SQL write-protection tested: Try `DROP TABLE` via chat (should error)
- [ ] Privacy notice displays below file uploader
- [ ] Test with read-only database user (for production)

---

## Common Questions

**Q: Can I increase the file size limit?**  
A: Yes, edit `MAX_UPLOAD_SIZE_MB = 20` in app.py (line 114). Larger uploads use more RAM.

**Q: Why can't I use my main database user?**  
A: SQL injection payloads might generate harmful queries. A read-only user cannot execute `DROP`, `DELETE`, etc. even if the LLM generates them.

**Q: Do I need RestrictedPython installed?**  
A: Strongly recommended! Without it, unsafe `exec()` is used as fallback. Always install via `pip install RestrictedPython`.

**Q: My private data is being sent to APIs — how do I run locally?**  
A: Run the app on your own infrastructure with your own LLM API keys. Consider using local models (Ollama, LM Studio) instead of cloud APIs.

**Q: Can I disable the privacy notice?**  
A: Technically yes (edit lines 119-123), but don't in production — users need to know!

---

## Security Resources

- **RestrictedPython docs:** https://restrictedpython.readthedocs.io/
- **OWASP: Code Injection:** https://owasp.org/www-community/attacks/Code_Injection
- **OWASP: SQL Injection:** https://owasp.org/www-community/attacks/SQL_Injection
- **OWASP: XSS Prevention:** https://owasp.org/www-community/attacks/xss/#prevention
- **Streamlit Security Best Practices:** https://docs.streamlit.io/deployment/streamlit-community-cloud/deploy-your-app#security

---

**Last Updated:** May 2024  
**DocBot Version:** 1.0 (Security Hardened)
