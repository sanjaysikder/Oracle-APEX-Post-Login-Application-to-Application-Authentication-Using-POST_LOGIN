# Oracle APEX Application-to-Application Authentication Using POST_LOGIN

## Overview

This document describes a secure application forwarding mechanism between Oracle APEX applications using a callback page and `APEX_AUTHENTICATION.POST_LOGIN`.

The objective is to allow a user authenticated in one Oracle APEX application to access another Oracle APEX application without manually entering credentials again.

---

# Architecture

```text
+-------------------+
| Source Application|
|  (Inventory)      |
+---------+---------+
          |
          |
          | Generate Secure URL
          v
+-------------------+
| Callback Page     |
| Target App        |
+---------+---------+
          |
          | Validate User
          | Verify Password Hash
          v
+-------------------+
| POST_LOGIN        |
| Create APEX       |
| Session           |
+---------+---------+
          |
          |
          v
+-------------------+
| Home Page         |
| Target App        |
+-------------------+
```

---

# Components

## Source Application

The source application generates a URL containing:

* User Identifier
* Encrypted Password Hash

Example:

```sql
SELECT
    'https://schoolerp.akijinsaf.com/scl/r/scl_inv/akij-insaf_scl-inventory/callback?userid='
    || CUSTOM_HASH('555976')
    || '&password='
    || CUSTOM_HASH('555976') AS URL
FROM dual;
```

---

## Callback Page

The callback page receives:

```text
userid
password
```

Example URL:

```text
https://schoolerp.akijinsaf.com/scl/r/scl_inv/akij-insaf_scl-inventory/callback
    ?userid=XXXXXXXX
    &password=YYYYYYYY
```

The callback page performs:

1. User Validation
2. Password Verification
3. Session Creation
4. Redirection

---

# Authentication Flow

## Step 1: Receive Parameters

```plsql
:USERID
:PASSWORD
```

---

## Step 2: Validate User

```plsql
SELECT PIN
INTO vUserInfo
FROM EMP_INFO
WHERE EMP_STATUS = 1
AND UPPER(EMP_ID) = UPPER(:USERID);
```

---

## Step 3: Verify Credentials

```plsql
IF vUserInfo = :PASSWORD THEN
```

---

## Step 4: Create APEX Session

Oracle APEX recommended authentication API:

```plsql
APEX_AUTHENTICATION.POST_LOGIN(
    p_username           => UPPER(:USERID),
    p_password           => :PASSWORD,
    p_uppercase_username => TRUE
);
```

This API:

* Creates a valid APEX session
* Registers the authenticated user
* Executes application authentication logic
* Initializes application security context

---

## Step 5: Redirect User

```plsql
APEX_UTIL.REDIRECT_URL(
    p_url =>
        'https://schoolerp.akijinsaf.com/scl/r/scl_inv/akij-insaf_scl-inventory/home?session='
        || :APP_SESSION
);
```

---

# Complete Callback Process

```plsql
DECLARE
    vUserInfo VARCHAR2(100);
BEGIN

    SELECT PIN
    INTO vUserInfo
    FROM EMP_INFO
    WHERE EMP_STATUS = 1
    AND UPPER(EMP_ID) = UPPER(:USERID);

    IF vUserInfo = :PASSWORD THEN

        APEX_AUTHENTICATION.POST_LOGIN(
            p_username           => UPPER(:USERID),
            p_password           => :PASSWORD,
            p_uppercase_username => TRUE
        );

        APEX_UTIL.REDIRECT_URL(
            p_url =>
                'https://schoolerp.akijinsaf.com/scl/r/scl_inv/akij-insaf_scl-inventory/home?session='
                || :APP_SESSION
        );

    ELSE

        APEX_UTIL.REDIRECT_URL(
            p_url =>
                'https://schoolerp.akijinsaf.com/scl/r/scl_inv/akij-insaf_scl-inventory/login'
        );

    END IF;

EXCEPTION

    WHEN NO_DATA_FOUND THEN

        APEX_UTIL.REDIRECT_URL(
            p_url =>
                'https://schoolerp.akijinsaf.com/scl/r/scl_inv/akij-insaf_scl-inventory/login'
        );

END;
```

---

# Source Application URL Generation

```plsql
DECLARE
    v_user        VARCHAR2(100);
    v_pass        VARCHAR2(100);
    v_hashed_user VARCHAR2(200);
    v_hashed_pass VARCHAR2(200);
    v_url         VARCHAR2(4000);
BEGIN

    v_user := :APP_USER;
    v_pass := :G_PASSWORD;

    v_hashed_user := CUSTOM_HASH(v_user);
    v_hashed_pass := CUSTOM_HASH(v_pass);

    v_url :=
        'https://schoolerp.akijinsaf.com/scl/r/scl_inv/akij-insaf_scl-inventory/callback?userid='
        || v_hashed_user
        || '&password='
        || v_hashed_pass;

END;
```

---

# Oracle Recommended Security Enhancements

## Do Not Pass Plain Passwords

Never send:

```text
userid=555976
password=555976
```

Use:

```text
userid=<encrypted value>
password=<encrypted value>
```

---

## Use HTTPS Only

All callback URLs must use TLS/SSL:

```text
https://
```

Never use:

```text
http://
```

---

## Add Token Expiration

Generate a temporary authentication token:

```text
TOKEN
USER_ID
CREATED_DATE
EXPIRY_DATE
```

Recommended validity:

```text
1-5 minutes
```

---

## Use One-Time Tokens

After successful authentication:

```sql
UPDATE SSO_TOKEN
SET USED_FLAG = 'Y'
WHERE TOKEN = :TOKEN;
```

---

## Validate Source Application

Store trusted application identifiers.

Example:

```text
INVENTORY
SCM
HR
ACCOUNTS
```

Reject requests from unknown sources.

---

## Audit Logging

Maintain an authentication log.

Example Table:

```sql
CREATE TABLE APP_SSO_LOG
(
    LOG_ID          NUMBER,
    USER_ID         VARCHAR2(100),
    SOURCE_APP      VARCHAR2(100),
    LOGIN_TIME      DATE,
    CLIENT_IP       VARCHAR2(100),
    STATUS          VARCHAR2(20)
);
```

---

# Recommended Production Architecture

Instead of:

```text
UserID + Password
```

Use:

```text
JWT Token
```

or

```text
One-Time SSO Token
```

Flow:

```text
Application A
      |
      |
Generate One-Time Token
      |
      v
Application B Callback
      |
Validate Token
      |
POST_LOGIN
      |
Home Page
```

This approach follows Oracle APEX security best practices and eliminates password transmission between applications.

---

# Oracle APIs Used

| API                            | Purpose                      |
| ------------------------------ | ---------------------------- |
| APEX_AUTHENTICATION.POST_LOGIN | Create authenticated session |
| APEX_UTIL.REDIRECT_URL         | Redirect user                |
| APP_SESSION                    | Current APEX session         |
| APP_USER                       | Current logged-in user       |

---

# Conclusion

For Oracle APEX application-to-application authentication:

* Use `APEX_AUTHENTICATION.POST_LOGIN`
* Use HTTPS
* Avoid transmitting passwords
* Prefer one-time SSO tokens
* Implement audit logging
* Add token expiry validation
* Redirect only after successful authentication

This architecture provides a secure and scalable mechanism for Oracle APEX Single Sign-On between multiple enterprise applications.
