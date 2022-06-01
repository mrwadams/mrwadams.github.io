---
layout: post
title: "OAuth 2.0 and OpenID Connect        "
date: 2022-06-01 00:00:00 -0000
categories:
---

Here are some notes I made while watching Nate Barbettini's presentation on OAuth 2.0 and OpenID Connect. You can find the video of the talk [here](https://www.youtube.com/watch?v=996OiexHze0).

## Delegated Authorisation
OAuth was designed to solve the problem of delegated authorisation (i.e. "How can I share my data with a website without giving it my password?").

Delegated authorisation allows a website / service to access data from another site (one that you're already using) within the bounds of a specified set of permissions (e.g. "allow access to public profile and contacts").

## OAuth Authorisation Flow
![OAuth 2.0 Authorisation Flow](/assets/images/oauth-flow.png)

Permissions are set using Scopes. The Authorisation Server has a list of scopes that it understands and can be requested by the Client.

The Back Channel is used to keep sensitive data out of the browser (Front Channel). A Back Channel needs to be created in advance to link the Client to the Authorisation Server.

## Authentication
Pre-2014, OAuth was being used for authentication **and** authorisation. 

OAuth wasn't designed to be used for authentication and so doesn't have a standard way to get the user's information. This resulted in every implementation being a little different.

## OpenID Connect
OpenID Connect (OIDC) was created as an extension to OAuth 2.0 to provide authentication.
![OpenID Connect](/assets/images/oidc.png)

OIDC provides:
- ID token
- UserInfo endpoint for getting more user information
- Standard set of Scopes
- Standardised implementation

Authorisation Server returns Access Token and ID Token (JWT).

Tokens are validated through Introspection.
