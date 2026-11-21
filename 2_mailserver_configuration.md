# OpenIDM 7.1.2 Outbound Email Configuration Guide (With Mailtrap)

This guide covers the complete end to end process for configuring outbound SMPT email in OpenIDM7.1.2 using Mailtrap.io, Including environment prerequisites, SMTP settings, and an automated onCreate script hook that triggers welcome emails whenever a user is created.

# 1. Prerequisites
- JDK 17
  - Ensure JAVA_HOME points to JDK 17
  - Ensure %JAVA_HOME%\bin is included in system PATH.
- Compiled OpenIDM Distribution: Run OpenIDM from an extracted binary distribution folder (eg. C:\u01\app\openidm\) rather than the raw source tree

- Mailtrap Account: A Mailtrap account with either an Email Sandbox( Dev/ QA Testing) or a verified Sending Domain (For production)



# 2. Obtain Mailtrap SMTP Credentials

Step 1. Configure mailtrap.io
- Navigate to mailtrap.io and signup with google
- Navigate to sandboxes and select it.
- Locate host, port, username, password under integration tab. 

Parameter	Mailtrap Email Sandbox (Dev / QA)	Mailtrap Live Sending (Production)
Host	sandbox.smtp.mailtrap.io	live.smtp.mailtrap.io
Port	2525 (or 587, 465)	587
Username	Your Sandbox inbox username	api
Password	Your Sandbox inbox password	Your Mailtrap API token
Sender Address	Any valid format (e.g., openidm@example.com)	Must match your verified domain


# 3. Configure Outbound Email in OpenIDM

Option A. Configure via Admin UI

1. Start OpenIDM by running startup.bat
2. Log into the Admin UI (http://localhost:8080/admin) or (https://localhost:8443/admin).
3. Use credentials openidm-admin/ openidm-admin
4. Navigate to configure -> System Preferences -> Email
5. Enter your SMTP parameters.
    - Enable email: Toggle ON
    - Host: sandbox.smtp.mailtrap.io
    - Port: 2525
    - Sender Email Address: openidm@example.com
    - Use SMTP authentication: Toggle ON
    - Username: Mailtrap username
    - Password: Mailtrap password
  
6. Use STARTTLS: Toggle ON
7. Click Save. OpenIDM automatically generates conf/external.email.json and encrypts the password.

Option B. Configure via conf/external.email.json (File Based)

1. Create c:\u01\app\openidm\conf\external.email.json.
2. Add the following JSON (replacing the credentials with your mailtrap values).

```
{
    "host" : "sandbox.smtp.mailtrap.io",
    "port" : "2525",
    "auth" : {
        "enable" : true,
        "username" : YOUR_MAILTRAP_USERNAME,
        "password" : YOUR_MAILTRAP_PASSWORD
    },
    "starttls" : {
        "enable" : true
    },
    "from" : "openidm@example.com",
    "debug" : true
}

```

3. Save the file. OpenIDM detects the file within a few seconds and encrypts the password field using it's $crypto service.


# 4. Configure Automated Welcome Emails on User Creation

Note: OpenIDM 7.1.2 bundles both javax.mail-1.4.7 and jakarta.mail-2.0.2, which can cause a NoSuchMethodError in the built-in external/email service. Invoking Packages.jakarta.mail.* directly inside your OpenIDM Rhino script bypasses this OSGI package collision without modifying any JAR files.

1. Create the Email script (sendWelcomeEmail.js) inside folder C:\u01\app\openidm\script\user\

```


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
            "<p>Your account (<strong>" + object.userName + "</strong>) has been successfully created.</p>",
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

2. Register the Hook in conf/managed.json

- Open conf/managed.json
- locate the "user" object inside the "objects" array
- Append require('user/sendWelcomeEmail'); to the existing "onCreate" "source" string (do not add a separate "file" key alongside "source")


```
{
    "name" : "user",
    "onCreate" : {
        "type" : "text/javascript",
        "source" : "require('ui/onCreateUser').setDefaultFields(object);require('ui/onCreateUser').createIdpPreferences(object);require('user/sendWelcomeEmail');"
    },
    ...
}
```

- Save conf/managed.json. OpenIDM automatically reloads the updated configuration without a restart.
  
# 5. Verification and Troubleshooting

1. Log into OpenIDM admin ui (http://localhost:8080/admin)
2. Go to Manage -> User -> + New User
3. Fill in the required fields (Ensuring email is populated) and click save.
4. Check your OpenIDM console output.
5. Open Mailtrap -> Sandboxes -> YOUR SANDBOX  and Confirm the HTML welcome email appears in your inbox.



