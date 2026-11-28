# OpenIDM 7.1.2 End-to-End Guide: Self-Service Registration, Email Verification & Welcome Emails

This guide documents the complete configuration for **OpenIdentityPlatform OpenIDM 7.1.2** using **Mailtrap**, covering environment prerequisites, custom registration attributes, a Jakarta Mail custom endpoint for self-service verification, and an automated `onCreate` welcome email hook.

---

## Table of Contents

1. [Environment Prerequisites](#1-environment-prerequisites)
2. [Configure Outbound SMTP Settings (`conf/external.email.json`)](#2-configure-outbound-smtp-settings-confexternalemailjson)
3. [Add a Custom Attribute (`department`) to `managed/user`](#3-add-a-custom-attribute-department-to-manageduser)
4. [Expose the Custom Attribute on the Registration UI](#4-expose-the-custom-attribute-on-the-registration-ui)
5. [Create a Custom Jakarta Mail Endpoint (`endpoint/sendEmail`)](#5-create-a-custom-jakarta-mail-endpoint-endpointsendemail)
6. [Configure Access Control & Self-Service Registration](#6-configure-access-control--self-service-registration)
7. [Configure the Automated Welcome Email (`onCreate` Hook)](#7-configure-the-automated-welcome-email-oncreate-hook)
8. [Clear Cache & Verify the Complete Flow](#8-clear-cache--verify-the-complete-flow)
9. [Troubleshooting Reference](#9-troubleshooting-reference)

---

## 1. Environment Prerequisites

* **Java Version:** **JDK 17 (LTS)** is required for OpenIDM 7.1.2 (`osgi.ee; version=17`). Older versions (JDK 11) fail to resolve core OpenIDM bundles, while newer versions (JDK 21+) break legacy mail packages.
* **Environment Variables:** Ensure `JAVA_HOME` points to your JDK 17 directory and `%JAVA_HOME%\bin` is included in your system `PATH`.
* **Active Configuration Directory:** Always place configuration and script files inside the root **`openidm/conf/`** and **`openidm/script/`** folders (not `openidm/samples/`) unless launching `startup.bat` with `-p`.

---

## 2. Configure Outbound SMTP Settings (`conf/external.email.json`)

Store your Mailtrap SMTP credentials centrally so OpenIDM encrypts the password automatically.

1. Create or update **`openidm/conf/external.email.json`**:

```json
{
    "host" : "sandbox.smtp.mailtrap.io",
    "port" : "2525",
    "auth" : {
        "enable" : true,
        "username" : "YOUR_MAILTRAP_USERNAME",
        "password" : "YOUR_MAILTRAP_PASSWORD"
    },
    "starttls" : {
        "enable" : true
    },
    "from" : "openidm@example.com"
}
```

2. Save the file. OpenIDM detects the file within a few seconds and encrypts the `"password"` field automatically using its `$crypto` service.

---

## 3. Add a Custom Attribute (`department`) to `managed/user`

To collect custom attributes during registration, define them in the `managed/user` schema first.

1. Open **`openidm/conf/managed.json`** (or configure via **Admin UI → Configure → Managed Objects → User**).
2. Add the **`department`** property under `"objects"` → `"user"` → `"schema"` → `"properties"`:

```json
"department" : {
    "title" : "Department",
    "type" : "string",
    "viewable" : true,
    "searchable" : true,
    "userEditable" : true
}
```

3. Add `"department"` to the `"order"` array inside `"schema"` so it appears in the desired order in the Admin UI:

```json
"order" : [
    "_id",
    "userName",
    "password",
    "givenName",
    "sn",
    "mail",
    "department",
    "accountStatus"
]
```

4. Save **`openidm/conf/managed.json`**.

---

## 4. Expose the Custom Attribute on the Registration UI

OpenIDM 7.1.2 renders the self-service registration screen (`#register/`) from static Handlebars/HTML templates.

1. Open **`openidm/ui/selfservice/default/templates/user/process/registration/userDetails-initial.html`**.
2. Locate the `<div class="form-group">` block for **Email** (`name="user.mail"`) and add a matching block for **Department** below it:

```html
<div class="form-group">
    <label class="sr-only separator" for="input-department">
        <span>Department</span>
    </label>
    <input type="text" placeholder="Department" id="input-department" name="user.department" class="form-control input-lg" />
</div>
```

3. Save **`userDetails-initial.html`**.

---

## 5. Create a Custom Jakarta Mail Endpoint (`endpoint/sendEmail`)

OpenIDM 7.1.2 bundles both `javax.mail-1.4.7` and `jakarta.mail-2.0.2`, which causes a `NoSuchMethodError` (`PropUtil`) in the built-in `external/email` service. Creating a custom Rhino endpoint using `Packages.jakarta.mail.*` bypasses this conflict cleanly without modifying JAR files.

### Step 5.1: Create the Endpoint Script (`script/endpoint/sendEmail.js`)

Create **`openidm/script/endpoint/sendEmail.js`**:

```javascript
/* global request, openidm, logger, Packages */

if (request.method === "action" && request.action === "send") {
    var emailData = request.content;
    var currentThread = Packages.java.lang.Thread.currentThread();
    var originalClassLoader = currentThread.getContextClassLoader();

    try {
        var mailConfig = openidm.read("config/external.email");
        var host = mailConfig.host || "sandbox.smtp.mailtrap.io";
        var port = String(mailConfig.port || "2525");
        var username = mailConfig.auth.username;
        var password = openidm.decrypt(mailConfig.auth.password);
        var fromAddr = emailData.from || mailConfig.from || "openidm@example.com";

        var props = new Packages.java.util.Properties();
        props.put("mail.smtp.host", host);
        props.put("mail.smtp.port", port);
        props.put("mail.smtp.auth", "true");
        props.put("mail.smtp.starttls.enable", "true");
        props.put("mail.smtp.ssl.trust", host);

        var session = Packages.jakarta.mail.Session.getInstance(props);
        currentThread.setContextClassLoader(session.getClass().getClassLoader());

        var msg = new Packages.jakarta.mail.internet.MimeMessage(session);
        msg.setFrom(new Packages.jakarta.mail.internet.InternetAddress(fromAddr));
        msg.setRecipients(
            Packages.jakarta.mail.Message.RecipientType.TO,
            Packages.jakarta.mail.internet.InternetAddress.parse(emailData.to)
        );
        msg.setSubject(emailData.subject || "Verify Your Email");
        msg.setText(emailData.body || "", "utf-8", "html");

        var transport = session.getTransport("smtp");
        transport.connect(host, parseInt(port, 10), username, password);
        transport.sendMessage(msg, msg.getAllRecipients());
        transport.close();

        logger.info("Self-service verification email sent to {}", emailData.to);
        ({ "status": "OK" });
    } catch (e) {
        logger.error("Failed to send self-service email: {}", String(e));
        throw { "code": 500, "message": String(e) };
    } finally {
        currentThread.setContextClassLoader(originalClassLoader);
    }
} else {
    throw { "code": 400, "message": "Unsupported request" };
}
```

> **Note:** Do not use a top-level `return` statement in OpenIDM Rhino endpoint scripts; evaluating `({ "status": "OK" });` as the final expression returns the JSON response.

### Step 5.2: Register the Endpoint Configuration (`conf/endpoint-sendEmail.json`)

Create **`openidm/conf/endpoint-sendEmail.json`**:

```json
{
    "context" : "endpoint/sendEmail",
    "type" : "text/javascript",
    "file" : "endpoint/sendEmail.js"
}
```

---

## 6. Configure Access Control & Self-Service Registration

### Step 6.1: Allow Access in `script/access.js`

Open **`openidm/script/access.js`** and add two rules inside the `configs` array so anonymous self-service flows can invoke `endpoint/sendEmail` and read `config/external.email`:

```javascript
        {
            "pattern"     : "endpoint/sendEmail",
            "roles"       : "*",
            "methods"     : "action",
            "actions"     : "send"
        },
        {
            "pattern"     : "config/external.email",
            "roles"       : "*",
            "methods"     : "read",
            "actions"     : "*",
            "customAuthz" : "!isExternalRequest()"
        },
```

### Step 6.2: Point Self-Service Registration to `endpoint/sendEmail`

1. Enable **User Registration** and **Email Validation** in the Admin UI (**Configure → Self-Service → User Registration**).
2. Open **`openidm/conf/selfservice-registration.json`** and update `"emailServiceUrl"` inside the `"emailValidation"` stage:

```json
        {
            "name" : "emailValidation",
            "identityEmailField" : "mail",
            "emailServiceUrl" : "endpoint/sendEmail"
        }
```

---

## 7. Configure the Automated Welcome Email (`onCreate` Hook)

### Step 7.1: Create the Welcome Email Script (`script/user/sendWelcomeEmail.js`)

Create **`openidm/script/user/sendWelcomeEmail.js`**:

```javascript
/* global object, openidm, logger, Packages */

if (object.mail) {
    var currentThread = Packages.java.lang.Thread.currentThread();
    var originalClassLoader = currentThread.getContextClassLoader();

    try {
        var mailConfig = openidm.read("config/external.email");
        var host = mailConfig.host || "sandbox.smtp.mailtrap.io";
        var port = String(mailConfig.port || "2525");
        var username = mailConfig.auth.username;
        var password = openidm.decrypt(mailConfig.auth.password);
        var fromAddr = mailConfig.from || "openidm@example.com";

        var props = new Packages.java.util.Properties();
        props.put("mail.smtp.host", host);
        props.put("mail.smtp.port", port);
        props.put("mail.smtp.auth", "true");
        props.put("mail.smtp.starttls.enable", "true");
        props.put("mail.smtp.ssl.trust", host);

        var session = Packages.jakarta.mail.Session.getInstance(props);
        currentThread.setContextClassLoader(session.getClass().getClassLoader());

        var msg = new Packages.jakarta.mail.internet.MimeMessage(session);
        msg.setFrom(new Packages.jakarta.mail.internet.InternetAddress(fromAddr));
        msg.setRecipients(
            Packages.jakarta.mail.Message.RecipientType.TO,
            Packages.jakarta.mail.internet.InternetAddress.parse(object.mail)
        );
        msg.setSubject("Welcome to OpenIDM, " + (object.givenName || object.userName) + "!");
        msg.setText(
            "<h3>Welcome, " + (object.givenName || object.userName) + "!</h3>" +
            "<p>Your account (<strong>" + object.userName + "</strong>) has been successfully created.</p>" +
            (object.department ? "<p>Department: <strong>" + object.department + "</strong></p>" : ""),
            "utf-8",
            "html"
        );

        var transport = session.getTransport("smtp");
        transport.connect(host, parseInt(port, 10), username, password);
        transport.sendMessage(msg, msg.getAllRecipients());
        transport.close();

        logger.info("Welcome email sent successfully to {}", object.mail);
    } catch (e) {
        logger.error("Failed to send welcome email to {}: {}", object.mail, String(e));
    } finally {
        currentThread.setContextClassLoader(originalClassLoader);
    }
}
```

### Step 7.2: Register the Hook in `conf/managed.json`

In **`openidm/conf/managed.json`**, append `require('user/sendWelcomeEmail');` to the existing `"onCreate"` `"source"` string under `"user"`:

```json
"onCreate" : {
    "type" : "text/javascript",
    "source" : "require('ui/onCreateUser').setDefaultFields(object);require('ui/onCreateUser').createIdpPreferences(object);require('user/sendWelcomeEmail');"
}
```

---

## 8. Clear Cache & Verify the Complete Flow

1. **Refresh OSGi Cache (when registering new endpoint scripts):**
   * Stop OpenIDM (`shutdown`), delete the **`openidm/felix-cache`** folder, and run **`startup.bat`**.
2. **Submit Registration:**
   * Open **`http://localhost:8080/#register/`** (with browser cache cleared via `Ctrl + F5`).
   * Fill in the user details including **Department** and click **Save**.
3. **Verify Email Link:**
   * Open your **Mailtrap Sandbox** and click the verification link in the registration email.
4. **Confirm Account & Welcome Email:**
   * OpenIDM displays *"Your account has been successfully registered"*, creates the user in **Manage → User** with the **Department** attribute saved, and delivers the HTML welcome email to Mailtrap.

---

## 9. Troubleshooting Reference

| Symptom / Error | Cause | Resolution |
| :--- | :--- | :--- |
| **`osgi.ee ... version=17` on startup** | Running OpenIDM 7.1.2 on Java 11 or an older JDK. | Install **JDK 17**, update `JAVA_HOME`, delete `felix-cache`, and run `startup.bat`. |
| **`NoSuchMethodError: PropUtil`** | Collision between `javax.mail-1.4.7` and `jakarta.mail-2.0.2`. | Route mail sending through `Packages.jakarta.mail.*` in Rhino scripts (`sendEmail.js` and `sendWelcomeEmail.js`). |
| **`ScriptCompilationException: invalid return`** | Using a top-level `return` statement in an OpenIDM endpoint script. | Remove `return` and leave `({ "status": "OK" });` as the final expression. |
| **`535 5.7.0 Invalid credentials`** | Passing the raw `$crypto` object instead of a decrypted string. | Decrypt the password with `openidm.decrypt(mailConfig.auth.password)`. |
| **`Could not convert socket to TLS`** | OpenIDM's internal truststore lacks Mailtrap's root CA. | Add `props.put("mail.smtp.ssl.trust", host);` in your mail script. |
| **`Resource 'endpoint/sendEmail' not found`** | Endpoint config saved under `samples/` or OSGi cache stale after a script fix. | Place `endpoint-sendEmail.json` in `openidm/conf/`, delete `felix-cache`, and restart OpenIDM. |
