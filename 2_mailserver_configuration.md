# 1. Configurations

Step 1. Configure mailtrap.io
- Navigate to mailtrap.io and signup with google
- Navigate to sandboxes and select it.
- Locate host, port, username, password under integration tab. 

Step 2. Configure OpenIDM email
- Create a file external.email.json directly inside openidm/conf folder

```
{
    "host" : "sandbox.smtp.mailtrap.io",
    "port" : 2525,
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
- Navigate to Configure > system preferences > email
- Host: sandbox.smtp.mailtrap.io, Port: 2525, Sender Email Address: openidm@noreply.com
- Enable "Use SMTP Authentication"
- Enable "Starttls"
- Click Save

Step 3. Trigger Email
- Create a user and check in Mailtrap, whether the email is triggered or not.

