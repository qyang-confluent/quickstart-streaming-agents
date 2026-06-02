## 5. Notify Claimants via Email with Streaming Agents

In this final step, you’ll extend the fraud‑detection pipeline with a **Streaming Agent** that emails a status update for a subset of reviewed claims.

- You will send emails for **only 3 reviewed claims** to limit LLM and Zapier usage.  
- For lab simplicity, **all emails go to a single recipient address** (your email), but:
  - Each email is addressed in the body to the individual claimant (`applicant_name`).
  - The message includes the **claim details** (amounts, etc.) and the **determination** (approved / partial / docs requested / denied).

---

### 5.1 Create the Zapier MCP Connection

First, tell Flink how to reach your Zapier MCP server by creating a **CONNECTION**.

> **Note for workshops:** Your instructor will provide a Zapier MCP token for you.  
> If you are running this lab on your own, follow the Zapier setup guide at [Zapier Setup](./assets/pre-setup/Zapier-Setup.md) in this project to obtain your token.

In the Flink SQL workspace for this environment/cluster, run: 
```sql
CREATE CONNECTION IF NOT EXISTS `zapier-mcp-connection`
WITH (
  'type' = 'MCP_SERVER',
  'endpoint' = 'https://mcp.zapier.com/api/v1/connect',
  'token' = '<YOUR_ZAPIER_TOKEN_HERE>',
  'transport-type' = 'STREAMABLE_HTTP'
);
```
- Replace `<YOUR_ZAPIER_TOKEN_HERE>` with your token.

### 5.2 Create the Zapier Tool (gmail_send_email)

Next, expose the Gmail tool from that connection as a Flink **TOOL**.
```sql
CREATE TOOL zapier
USING CONNECTION `zapier-mcp-connection`
WITH (
'type' = 'mcp',
'allowed_tools' = 'gmail_send_email',
'request_timeout' = '30'
);
```
### 5.3 Create the Claim Status Email Agent

Now define an agent that:

1. Reads `EMAIL RECIPIENT`, `EMAIL SUBJECT`, and `EMAIL BODY TEMPLATE` from its prompt.  
2. Calls **only** `gmail_send_email` using those exact values.  
3. Returns a brief confirmation string.

This agent uses the same `llm_textgen_model` you already used for the fraud‑investigation agent in this lab.
```sql
CREATE AGENT `claims_status_email_agent`
USING MODEL `llm_textgen_model`
USING PROMPT '
You are a FEMA claims notification assistant.

You will receive:
- A short description of a single reviewed claim (claimant, amounts, verdict).
- Email instructions in this exact format:

EMAIL RECIPIENT: <recipient email>
EMAIL SUBJECT: <email subject>
EMAIL BODY TEMPLATE: <full email body text>

Your tasks:

1. Parse the prompt and extract exactly:
   - EMAIL RECIPIENT
   - EMAIL SUBJECT
   - EMAIL BODY TEMPLATE

2. Call the gmail_send_email tool once with:
   - to: EMAIL RECIPIENT as a single string (NOT an array)
   - subject: EMAIL SUBJECT
   - body: EMAIL BODY TEMPLATE

3. Do not modify these values. Use them exactly as provided.

4. After calling gmail_send_email, return a brief text confirmation including:
   - The claim_id (if present)
   - The applicant_name (if present)
   - The verdict
   - The recipient email used

Do NOT call any tools other than gmail_send_email.
'
USING TOOLS `zapier`
WITH (
  'max_iterations' = '5'
);
```

---

### 5.4 Send Emails for 3 Reviewed Claims

Finally, invoke the email agent on **only 3 records** from `claims_reviewed` to reduce token and email usage.

> **IMPORTANT**  
> Replace `<<YOUR-EMAIL-ADDRESS-HERE>>` below with the **the email address** where you want to receive the notification emails.
```sql
SET 'sql.state-ttl' = '14 d';

CREATE TABLE claims_status_emails
WITH ('changelog.mode' = 'append') AS
SELECT
  r.claim_id,
  r.applicant_name,
  r.verdict,
  r.claim_amount,
  r.damage_assessed,
  r.insurance_amount,
  r.summary,
  r.issues_found,
  agent_result.status   AS agent_status,
  agent_result.response AS agent_response
FROM (
  -- Limit to 3 reviewed claims to reduce token and email usage
  SELECT *
  FROM claims_reviewed
  LIMIT 3
) r,
LATERAL TABLE (
  AI_RUN_AGENT(
    `claims_status_email_agent`,
    CONCAT(
      'Reviewed FEMA claim status notification.', '\n',
      'Claim ID: ', r.claim_id, '\n',
      'Applicant Name: ', COALESCE(r.applicant_name, 'Unknown'), '\n',
      'Verdict: ', COALESCE(r.verdict, 'UNKNOWN'), '\n',
      'Claim Amount: $', COALESCE(CAST(r.claim_amount AS STRING), '0'), '\n',
      'Damage Assessed: $', COALESCE(CAST(r.damage_assessed AS STRING), '0'), '\n',
      'Insurance Amount: $', COALESCE(CAST(r.insurance_amount AS STRING), '0'), '\n',
      'Issues Found: ', COALESCE(r.issues_found, 'None – claim passes all checks.'), '\n',
      'Summary: ', COALESCE(r.summary, 'No additional summary available.'), '\n',
      '\n',
      -- Email control instructions for the agent
      'EMAIL RECIPIENT: <<YOUR-EMAIL-ADDRESS-HERE>> ',
      'EMAIL SUBJECT: FEMA Disaster Assistance Claim Status – Claim ', r.claim_id,
      ' (', COALESCE(r.verdict, 'UNKNOWN'), ') ',
      'EMAIL BODY TEMPLATE: ',
      'Dear ', COALESCE(r.applicant_name, 'Applicant'), ',\n\n',
      'We are writing to inform you of the status of your FEMA disaster assistance claim.\n\n',
      'Claim details:\n',
      '- Claim ID: ', r.claim_id, '\n',
      '- City: (see claim record)\n',
      '- Claim Amount: $', COALESCE(CAST(r.claim_amount AS STRING), '0'), '\n',
      '- Verified Damage: $', COALESCE(CAST(r.damage_assessed AS STRING), '0'), '\n',
      '- Insurance Coverage: $', COALESCE(CAST(r.insurance_amount AS STRING), '0'), '\n\n',
      'Determination:\n',
      '- Decision: ', COALESCE(r.verdict, 'UNKNOWN'), '\n',
      '- Explanation: ', COALESCE(r.issues_found, 'None – your claim passed all checks.'), '\n\n',
      'This email was generated automatically by a streaming fraud detection and notification system built on Confluent Intelligence.\n\n',
      'Sincerely,\n',
      'FEMA Disaster Assistance Program'
    ),
    r.claim_id,
    MAP['debug', 'true']
  )
) AS agent_result(status, response);
```

Then you can verify that up to **3 emails** were triggered and see the agent responses:
```sql
SELECT * FROM claims_status_emails;
```
