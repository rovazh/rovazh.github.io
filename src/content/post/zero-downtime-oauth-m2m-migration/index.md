---
title: "Changing OAuth Provider with Zero Downtime"
description: "This is an outline of a phased approach to OAuth provider migration"
publishDate: "21 March 2026"
updatedDate: "26 March 2026"
tags: ["migration"]
---

## Introduction

Recently, I completed a successful migration to a new OAuth provider in production. The provider authorized incoming requests sent by external clients to a critical service with a high SLA. Because of the service's criticality, the migration had to be performed with zero downtime to remain compliant with that SLA. Consequently, I developed and implemented a strategy to switch from one OAuth provider to another without service disruption. In the following post, I will walk you through the challenges of this migration and the decisions I made to overcome them.

## The Service

Before diving into the migration details, let me briefly describe the service and how it authorizes incoming requests. The service provides an API that clients integrate with their backends; these backends then make requests to the API on their own behalf. Since no human user is involved, this communication is referred to as Machine-to-Machine (M2M).

To ensure only authorized backends can call the service, an authorization mechanism is required. We implement this using the OAuth client credentials grant flow, which is well-suited for M2M communication. Simply put, a client's backend requests an access token from an authorization server and uses it to access the service. The service then verifies the access token and handles the request if the token is valid.

The OAuth provider's role is to host the authorization server and issue client credentials. These credentials are generated once a client registers their backend on a special web portal backed by the service. Clients do not access the provider directly; rather, they call a special proxy service that forwards the request to the actual authorization server.

## The Challenges

In the OAuth client credentials grant flow, credentials are static and remain valid for a long period. This implies that once a client registers their backend, the generated credentials are hardcoded in their configurations and remain intact until they expire. If a simple cutover had been attempted without thorough planning, every client would have been forced to register their backends with the new provider and update their configurations simultaneously. Such a disruptive change would have inevitably led to downtime.

## The Solution: A Three-Phase Approach

Considering these challenges, I prepared a migration strategy consisting of three distinct phases:
+ Phase One: Integration. Integrate the new OAuth provider into the service.
+ Phase Two: Rolling Out. Activate the new provider while keeping the legacy provider functional.
+ Phase Three: Cleanup. Deactivate and decommission the legacy provider.

### Phase One: Integration

To integrate the new provider, three system components required modification: the web portal, the proxy service, and the service's authorization logic.

Modifications to the web portal were straightforward. Originally, the portal called the legacy provider's API during registration. I implemented a compatible adapter for the new provider’s API, controlled by a feature flag to determine which API was in use.

![Wep Portal diagram](./web-portal-diagram.svg)

Crucially, credentials issued before the switch had to remain valid so clients could continue using the service without disruption. To make this possible, the other two components had to support both providers simultaneously.

In the proxy service, we added integration for the new provider. When a client sends a request to exchange credentials for a token, the Proxy extracts the Client ID to determine which provider the client belongs to.

![Proxy Service diagram](./proxy-service-diagram.svg)

Finally, the service's authorization logic was modified to support access tokens from both providers. By checking the iss (issuer) claim in the token, the service utilizes the appropriate SDK to verify it.

![Auth Logic diagram](./auth-logic-diagram.svg)

### Phase Two: Rolling Out

After development was complete, the components were gradually deployed to the upper environments. Once readiness was confirmed, the feature flag was enabled, and new client backends began registering with the new provider. Simultaneously, existing clients continued using credentials issued by the legacy provider until they expired.

### Phase Three: Cleanup

The dual-provider solution remained operational until the final legacy credentials expired. After confirming that no clients were still using the old provider, it was safe to remove the legacy integration from the system components. This final step marked the successful completion of the migration.
