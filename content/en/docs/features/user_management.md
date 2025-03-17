---
title: User Management
description: >
  Describes the way to manage users
weight: 9
---

LLMariner installs [Dex](https://github.com/dexidp/dex) by default. Dex is an identity service that uses [OpenID Connect](https://openid.net/developers/how-connect-works/) for authentication.

The Helm chart for Dex is located at <https://github.com/llmariner/rbac-manager/tree/main/deployments/dex-server>. It uses a [built-in local connector](https://dexidp.io/docs/connectors/local/) and has the following configuration by default:

``` yaml
staticPasswords:
- userID: 08a8684b-db88-4b73-90a9-3cd1661f5466
  username: admin
  email: admin@example.com
  # bcrypt hash of the string: $(echo password | htpasswd -BinC 10 admin | cut -d: -f2)
  hash: "$2a$10$2b2cU8CPhOTaGrs1HRQuAueS7JTT5ZHsHSzYiFPm1leZck7Mc8T4W"
```

You can switch a connector to an IdP in your environment (e.g., LDAP, GitHub). Here is an example connector configuration with Okta:

``` yaml
global:
  auth:
    oidcIssuerUrl: https://<LLMariner endpoint URL>/v1/dex

dex-server:
  oauth2:
    passwordConnector:
      enable: false
    responseTypes:
    - code
  connectors:
  - type: oidc
    id: okta
    name: okta
    config:
      issuer: <Okta issuer URL>
      clientID: <Client ID of an Okta application>
      clientSecret: <Client secret of an Okta application>
      redirectURI: https://<LLMariner endpoint URL>/v1/dex/callback
      insecureSkipEmailVerified: true
  enablePasswordDb: false
  staticPassword:
    enable: false
```

Please refer to the [Dec documentations](https://dexidp.io/docs/connectors/) for more details.

The Helm chart for Dex creates an ingress so that HTTP requests to `v1/dex` are routed to Dex. This endpoint URL works as the OIDC issuer URL that CLI and backend servers use.
