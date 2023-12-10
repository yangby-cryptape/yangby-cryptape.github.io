---
title: Setup a Non-Interactive "LESS SECURE" Mail Client for Gmail
date: 2019-06-10T08:56:21+08:00
categories: [productivity]
tags: [getmail, gmail]
---

Some command-line tools can't fetch mails from [Gmail], **since the
administrator had select the option - "disabled access to less secure apps
for all users"**.
<!--more-->

#### Enable APIs

- Create a project in [API Console].
- Fill the required fields for [OAuth Consent Screen].
- Create an `OAuth client ID` in the [Credentials Page], to obtain a `client_id`
  and a `client_secret`.

#### Configure the "LESS SECURE" Mail Client

For example, in [`getmail`]:

- Create a file `/a-path-to-the-credential/gmail.json` to store the token:

  ```json
  {
    "scope": "https://mail.google.com/",
    "user": "your-name@your-domain.com",
    "client_id": "your-client-id",
    "client_secret": "your-client-secret",
    "token_uri": "https://accounts.google.com/o/oauth2/token",
    "auth_uri": "https://accounts.google.com/o/oauth2/auth",
    "redirect_uri": "urn:ietf:wg:oauth:2.0:oob",
    "auth_provider_x509_cert_url": "https://www.googleapis.com/oauth2/v1/certs"
  }
  ```

- Get the initial access and refresh tokens:

  ```shell
  getmail-gmail-xoauth-tokens --init /a-path-to-the-credential/gmail.json
  ```

- Update the `getmailrc`:

  ```ini
  [retriever]
  type = SimpleIMAPSSLRetriever
  server = imap.gmail.com
  username = your-name@your-domain.com
  use_xoauth2 = True
  password_command = ("getmail-gmail-xoauth-tokens", "/a-path-to-the-credential/gmail.json")
  ```

#### References

- [Google Identity Platform: OAuth 2.0 for Mobile & Desktop Apps]
- [Google Identity Platform: Using OAuth 2.0 to Access Google APIs]
- [Enable `XOAUTH2` configuration for Gmail in `getmail`]

[Gmail]: https://mail.google.com
[API Console]: https://console.developers.google.com
[Credentials Page]: https://console.developers.google.com/apis/credentials
[OAuth Consent Screen]: https://console.developers.google.com/apis/credentials/consent
[`getmail`]: http://pyropus.ca/software/getmail
[Google Identity Platform: OAuth 2.0 for Mobile & Desktop Apps]: https://developers.google.com/identity/protocols/OAuth2InstalledApp
[Google Identity Platform: Using OAuth 2.0 to Access Google APIs]: https://developers.google.com/identity/protocols/OAuth2
[Enable `XOAUTH2` configuration for Gmail in `getmail`]: https://www.bytereef.org/howto/oauth2/getmail.html
