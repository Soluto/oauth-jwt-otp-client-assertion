---
title: Non-interactive OAuth 2.0 Client Authentication grant
docname: draft-seamless-flow-latest
ipr: trust200902
cat: info
pi:
  sortrefs: 'yes'
  strict: 'yes'
  symrefs: 'yes'
  toc: 'yes'
author:
- ins: O. Levi Hevroni
  name: Omer Levi Hevroni
  email: omerlh@gmail.com
  organization: Soluto by Asurion

--- abstract

This specification defines the use of a One Time Password, encoded as JSON Web Token (JWS) Bearer Token, as a means for requesting an OAuth 2.0 access token as well as for client authentication.

--- middle

# Introduction

## Motivation

Authentication is a crucial part of modern application. There are various authentication methods for client side applications, and all those methods requires user interaction (e.g. login). This is due to the fact that there is no secure way to embed credentials in the application code.

While asking the user to login in order to authenticate the app is a strong authentication solution, it has impact on the application behavior. 
A login is just another step the user has to complete in order to use the apps, which users don't always like to fulfill.

Also, there are cases for applications without any UI, for example - Internet of Things applications. For those applications, adding a login steps could be a challenge.

In this document, we propose an extension to OAuth 2.0 protocol that provides a new authentication grant dedicated for those cases. 
This grant will allow an application to use strong authentication solution without user interaction. 

This document defines how a One Time Password, encoded in a JWS, can be used to authenticate the client. 
In order for the client to perfom an authentication request, an initial registration step is required.
This registration step is not part of this protocl, and should be defined by the authorization server.

## Exisiting Solutions

There are alternatives to this protocol, and this section will discuss them.
All the interactive grants (authorization code, resource owner etc) will not be discussed.

### Client Credentials grant
This grnat (as defined in {{!RFC6749}}) allows applications to authenticate without user interaction.
It is intened to used by applications running on trusted environment.
Mobile applications are not running on trusted environment, and therfore should not use this grant.
See the Security section for discussion on the various threat and how this protocol mitigate them.
Also reffer to secion 10.1 in {{!RFC6749}}, which strongly advice against using this grant on native applications.

### Device grant
This grant is for Browserless and Input Constrained Devices. 
In this grant the login is performed on a different device, which could handle interactive login.
Therefore, it still requires user interaction, which this protocol aims to avoid.

### JWT Client Assertion
This grant (as degined in {{!RFC7523}}) could be used by mobile application for seamless authentication.
The grant used signed JWT (see {{!RFC7519}}) to authenticate the client. 
This has two disadvantages when compared with this protocol:
 - Siginificant part of the security of the protocol is the expiration date of the JWT.
In case an hacker was able to obtain a JWT, she will be able to perform authentication request until the JWT expires.
Therefore, it is advised to use as shorter expiration time as possible.
Time can be a challenge on mobile device, which are not always synchronized with the global time.
Usage of JWT would require the authorization server to allow very long JWT expiration time.
 - Detectining Compromised Signing Key.
 As discussed on the security section, this protocol allows the authorization server to detect compromised signing key.
 See the discussion there for refference.
This mitigation does not exist in JWT client assertion grant.
 - Replay Attack.
 JWT client assertion grant mitigate Replay Attack by setting the expiration date.
 As discussed previously on this section, time can be a challenge for mobile application.
 Therefore, using this grant without setting the expiration date of the JWT could be an option.
 In such case, the protocol is vulnerable to Replay Attack.

## Terminology

In this document, the key words "MUST", "MUST NOT", "REQUIRED",
"SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY",
and "OPTIONAL" are to be interpreted as described in {{!RFC2119}}.

# Note to Readers

> **Note to the RFC Editor:**  Please remove this section prior
> to publication.

Development of this draft takes place on Github at:  https://github.com/Soluto/oauth-JWS-otp-client-assertion

# HTTP Parameter Bindings for Transporting Assertions

The OAuth Assertion Framework {{!RFC7521}} defines generic HTTP parameters for transporting assertions (a.k.a. security tokens) during interactions with a token endpoint. This section defines specific parameters and treatments of those parameters for use with JWS Bearer Tokens.

## Using OTP JWS for client authentication

To use a OTP JWS, the client first need to generate the OTP as defined in section "JWS format and request processing". Than, the client need to use the following parameter values and encodings.

The value of the "client_assertion_type" is "urn:ietf:params:oauth:client-assertion-type:JWS-otp".

The value of the "client_assertion" parameter contains a single JWS, as defined in {{!RFC7515}}. It MUST NOT contain more than one JWS.

The following example demonstrates client authentication using a JWS during the presentation of an authorization code grant in an access token request (with extra line breaks for display purposes only):

~~~~~~~~~~
     POST /token.oauth2 HTTP/1.1
     Host: as.example.com
     Content-Type: application/x-www-form-urlencoded

     grant_type=token id_token&&
     client_assertion_type=urn%3Aietf%3Aparams%3Aoauth%3A
     client-assertion-type%3AJWS-otp&
     client_assertion=eyJhbGciOiJSUzI1NiIsImtpZCI6IjIyIn0.
     eyJpc3Mi[...omitted for brevity...].
     cC4hiUPo[...omitted for brevity...]
~~~~~~~~~~

# JWS format and request processing

## One Time Password generation

To generate one time password (OTP) as defined in {{!RFC2289}}, the client use it's state, created during the registration request, which is not covered in this document. 
The state consist from 2 numbers: `previous` and `next`. 
Each of those numbers can hold signed int, up to 64 bytes length.
In order to generate a new JWS, the client has to roll this payload. The rolling is done by setting the value of `previous` to the value of `current`, and setting new crypto random, as defined in {{!RFC4086}}, value to `next`. 
For example, assuming this is the current state of the app:

~~~~~~~~~~
previous: 1
next: 2
~~~~~~~~~~

After rolling, this will be the payload:

~~~~~~~~~~
previous: 2
next: 5
~~~~~~~~~~

## Creating the JWS
After rolling the payload, the client can create the JWS.
This is the format of the JWS payload:

~~~~~~~~~~
{
    previous: 2
    next: 5
    client-id: 89
}
~~~~~~~~~~

Where `client-id` is the id used when this client first registered.
All the fields are required. 
Any other fields besides those will be ignored.
To sign the JWS, the client use it's own key, which was genereated during the registration of this client. 

## Request processing
In order to issue an access token response as described in OAuth 2.0 {{!RFC6749}}, the authorization server MUST validate the JWS according to the criteria below. 
Application of additional restrictions and policy are at the discretion of the authorization server.
After decoding the JWS and extracting the `client-id`, the server will fetch:
 - The key correspond to this client, received on the registration request
 - The current state of this client, from the last successful request, or from the registration

The server verifies that the JWS is vallid, by using the client's key.
If the signature is valid, the server can validate the payload:
 - If the client's `previous` is equals to the server `new`, the request is valid. The server will issue a token, as specific in OAuth 2.0 {{!RFC6749}}
 - If the client `previous` equals to the server `previous`, and the client `next` equals to the server `next`, the server construct an error response as defined in OAuth 2.0 {{!RFC6749}}
 - Any other case will be threated by the server as an indication of a malicious attack, and should be reported accordenly. The server construct an error response as defined in OAuth 2.0 {{!RFC6749}}

# Security Considerations

This protocl was designed for mobile application.
The following sections will discuss threats which are relevants for mobile applications and are mitigated by this protocol.

## Replay Attacks
Due to the usage of OTP, a replay attack is not feasable. 
If an attacker will try to replay authentication request, an error response will return.
Also, because of how the OTP is generated, guessing it is almost impossible (see the OTP Generation section).
Reffer to the Request processing section for more details.

## Compromised Devices
As the application is running on a mobile device, an attacker can gain phisical access to the device.
In such a scenario, the attacker will be able to compromise it and retrieve the state and the signing key.
This will allows the attacker to impersonate the device and request an access token.
The attacker will be able to authenticate as until the first time the device will try to authenticate.
When the device will try to authenticate, the request will failed.
It will fail because the state on the authorization server will match the attacker's state, not the one on the device.

The device authentication request will revoke the request (see Request processing section).
This will cause both the device and the attacker to not be able to authentication.
In such cases, an alternative flow is required in order to allow the device to authenticate.
Such a flow is not part of this standard.

In order to this mitigation to be effective, the device must to perform authentication request occasionaly.
The period between authentication requests should be 24 hours or less, depend on the client.

## Man in the Middle
Performing Man in the Middle (MitM) attack on mobile application is relatively simple.
It is highly recommended to use TLS {{!RFC5246}} for all the requests in this protocl.
It is also recommened to implement Certificate Pinning for all the requests.
For more details, please reffer to this [guide](https://www.owasp.org/index.php/Certificate_and_Public_Key_Pinning) by OWASP.

## Reverse Engineering
The mobile application code is publicly available, which make reverse engineering a simple task.
This attack is irrelevant to this protocol.
No sensitive data should be embedded in the application code.
All that is required for the authentication request should be generated on the device.

## OTP Generation
The security of the OTP is as strong as the randomness used to generate it.
Only strong, secure random implementation (as described in {{!RFC4086}}) should be used.
Usage of weak random protocol will allows the attacker the guess the number, and by that generates the password herself.
The state (`next` and `new`) is not considered a secret. 
Compromise of state only, without the signing key, will not allows the attacker to perform authentiction request.
It is still advised to store them securely, and follow the operation system recomendation ([iOS](https://www.apple.com/business/docs/iOS_Security_Guide.pdf), [Android](https://developer.android.com/training/articles/security-tips.html#UserData)).

## Signing Key Consideration

### Generation and Storage

A foundemental part of the security of the protocl is the key used to sign the JWS.
The key should be generated and stored as securely as possible, and if possible to use the tools provided by the OS.
On iOS, use Key Chain to generate and persist the key.
On Andoird, the best option is the Key Store, but due to implementation limitations, it is advised to use OpenSSL and store the key as securley as possible.

### Algorithm
Asymetric encryption and signing algorithms are preffered over symetric ones.
The main advantages of such protocol is that the private key never leaves the device. 
Even if an attacker where able to capture the public (either in transit or by compromisng the authorization server), she will not be able to use it.
For any algorithm that is chosen, a strong enough key should be generated.
In case of RSA, 2048 bytes is the minimum key size.

# IANA Considerations

TODO IANA



--- back