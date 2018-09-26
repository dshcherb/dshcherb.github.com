---
layout: post
title: "Adding Multi-Factor Authentication Rules to Keystone Users via Custom Options"
date: 2018-09-26 18:40:00 +0300
description: Adding Multi-Factor Authentication Rules to Keystone Users via Custom Options
---

# Introduction

Keystone has a concept of [authentication plug-ins](https://docs.openstack.org/keystoneauth/latest/authentication-plugins.html) and there can be quite [a few of them](https://docs.openstack.org/keystoneauth/latest/authentication-plugins.html#loading-plugins-by-name) used separately for user authentication based on a method selected in API/CLI/UI.

When it comes to multi-factor authentication (MFA) it used to be that this could only be done via an identity provider's own MFA method which implied that you already use SAML or OIDC and forward a user to an external authentication service from which signed response contents are provided to Keystone. This approach effectively removes the credential validation part from Keystone and it only validates the result of an authentication.

# Native per-user MFA

There was some [work done for the Ocata cycle](https://blueprints.launchpad.net/keystone/+spec/per-user-auth-plugin-reqs) to enable support for multiple authentication methods on a per-user basis. Since Keystone by itself can use a database to perform password validation and other means of authentication (such as TOTP with Google Authenticator) it should be possible to use both methods for a single user. However, there needs to be some metadata associated with a user and logic to use that metadata for Keystone to decide which methods to use to authenticate a user. This is what [those changes](https://review.openstack.org/#/c/424334/) were about.

# Adding Options to Users

There is a special "options" resource stored in the Keystone database for a given user which is not documented in the Keystone API reference at the time of writing. This resource can store [MFA-related options](https://github.com/openstack/keystone/blob/stable/queens/keystone/identity/backends/resource_options.py#L79-L105
) such as multi_factor_auth_rules. The documented usage references are scarce but this one came up:

{% highlight bash %}
{% raw %}

user["options"]["multi_factor_auth_rules"] = [["password", "totp"], ["password", "custom-auth-method"]]

{% endraw %}
{% endhighlight bash %}

At the time of writing there is no way to specify custom options for new users or for user updates via OpenStack CLI or via Horizon. The only way is to use the Keystone API directly in an undocumented way:

{% highlight bash %}
{% raw %}

cat token-request.json 
{
    "auth": {
        "identity": {
            "methods": [
                "password"
            ],
            "password": {
                "user": {
                    "domain": {
                        "name": "admin_domain"
                    },
                    "name": "admin",
                    "password": "42"
                }
            }
        },
        "scope": {
            "project": {
                "domain": {
                    "name": "admin_domain"
                },
                "name": "admin"
            }
        }
    }
}

# obtain a token
curl -si -d @token-request.json -H "Content-type: application/json" http://keystone.openstack.example:5000/v3/auth/tokens | awk '/X-Subject-Token/ {print $2}'
9d9a12b0808643fb8facfe0d3c2ac217


cat user-mfa-create.json
{
    "user": {
        "default_project_id": "baa36135b15c4d3f9667dc03ddabc23f",
        "domain_id": "24eecdf281e544ba9d41bae91299ea5a",
        "enabled": true,
        "name": "jdoe",
        "password": "test",
        "description": "James Doe user",
        "options": { "multi_factor_auth_rules": [["password"]] }
    }
}

# use a token to create a user with MFA rules
curl -si -d @user-mfa-create.json -H "Content-type: application/json" -H"X-Auth-Token:9d9a12b0808643fb8facfe0d3c2ac217" http://keystone.openstack.example:5000/v3/users
HTTP/1.1 201 Created
Date: Wed, 26 Sep 2018 12:30:40 GMT
Server: Apache/2.4.18 (Ubuntu)
Vary: X-Auth-Token
X-Distribution: Ubuntu
x-openstack-request-id: req-44494d53-92fc-4c26-b2c8-75320f3b6fab
Content-Length: 400
Content-Type: application/json

{"user": {"description": "James Doe user", "name": "jdoe", "domain_id": "24eecdf281e544ba9d41bae91299ea5a", "enabled": true, "links": {"self": "http://keystone.openstack.example:5000/v3/users/dd3d4a97de1b43428b50566be8f76296"}, "options": {"multi_factor_auth_rules": [["password"]]}, "default_project_id": "baa36135b15c4d3f9667dc03ddabc23f", "id": "dd3d4a97de1b43428b50566be8f76296", "password_expires_at": null}}


# the options field now contains the relevant array of MFA rules which can be extended to have other methods like totp

openstack user show jdoe
+---------------------+---------------------------------------------+
| Field               | Value                                       |
+---------------------+---------------------------------------------+
| default_project_id  | baa36135b15c4d3f9667dc03ddabc23f            |
| description         | James Doe user                              |
| domain_id           | 24eecdf281e544ba9d41bae91299ea5a            |
| enabled             | True                                        |
| id                  | dd3d4a97de1b43428b50566be8f76296            |
| name                | jdoe                                        |
| options             | {'multi_factor_auth_rules': [['password']]} |
| password_expires_at | None                                        |
+---------------------+---------------------------------------------+

{% endraw %}
{% endhighlight bash %}


This could be used to combine password authentication with TOTP or other authentication methods. Using SAML with TOTP also comes to mind if there is no MFA configured on the identity provider side.
