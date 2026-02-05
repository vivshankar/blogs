**TL;DR:** As AI agents increasingly collaborate through the Agent2Agent (A2A) protocol, proper authorization becomes critical to ensure trust, privacy, and security. Without robust authorization, agents could access sensitive data without consent, perform unauthorized actions, or create security vulnerabilities in multi-agent systems.

This article addresses four key authorization challenges:

1. **Trust & Identity**: Verifying which agents are making requests and maintaining clear delegation chains
2. **User Privacy**: Ensuring agents only access data with proper consent and human-in-the-loop controls
3. **Fine-Grained Access**: Limiting agents to specific resources and actions they're permitted to perform
4. **Security**: Implementing least privilege principles and containing potential breaches

Learn how to implement standards-based OAuth 2.0 authorization patterns using IBM Verify for three types of agents: conversational agents (user-initiated), autonomous agents acting on behalf of users, and fully autonomous system agents. Each pattern includes detailed flows, implementation examples, and security considerations.

# Agent2Agent Authorization using IBM Verify

The [Agent2Agent (A2A) protocol](https://a2a-protocol.org/) is a communication protocol for AI agents, initially introduced by Google in April 2025. This open protocol is designed for multi-agent systems, allowing interoperability between AI agents from varied providers or those built using different AI agent frameworks.

This article assumes an understanding of the Agent2Agent specification and the components, like the A2A client and server, Agent card, task, etc. For a primer on the A2A protocol, refer to the [A2A specification](https://a2a-protocol.org/specification/) or this article [published by IBM](https://www.ibm.com/think/topics/agent2agent-protocol).

## The Critical Need for Authorization in A2A Systems

As AI agents increasingly communicate and collaborate through protocols like Agent2Agent (A2A), authorization becomes paramount for several critical reasons:

1. **Trust and Identity Verification**
   - **Authentication**: A2A flows must verify the identity of calling agents to ensure only legitimate agents can request tasks
   - **Delegation Chains**: When agents act on behalf of users or other agents, maintaining a clear chain of trust is essential
   - **Audit Trails**: Every action must be traceable to understand who (user or agent) initiated what action and when

2. **User Privacy and Consent**
   - **Human-in-the-Loop**: When agents act on behalf of users, explicit or implicit consent is required before accessing sensitive data
   - **Consent Management**: Users must be able to grant, review, and revoke permissions for agents accessing their resources
   - **Data Protection**: Authorization ensures agents only access data they're permitted to see, protecting user privacy

3. **Fine-Grained Access Control**
   - **Scope-Based Permissions**: Different agents require different levels of access (e.g., read-only vs. read-write)
   - **Resource-Specific Access**: Authorization can limit agents to specific resources (e.g., only work calendar, not personal)
   - **Action-Based Controls**: Permissions can be tied to specific actions agents are allowed to perform

4. **Security and Risk Mitigation**
   - **Least Privilege Principle**: Agents should only have the minimum permissions necessary to complete their tasks
   - **Breach Containment**: If an agent is compromised, proper authorization limits the scope of potential damage
   - **Anomaly Detection**: Authorization systems can detect and prevent unusual or suspicious agent behavior

## Key Considerations for A2A Authorization

1. **Agent Type and Context**: Different agent types require different authorization approaches:
   - **Conversational Agents**: Invoked by user prompts, have access to user tokens, need to exchange for appropriate permissions
   - **Autonomous Agents (User Context)**: Run in background on behalf of users, may need CIBA or refresh tokens for consent
   - **Autonomous Agents (System Context)**: Operate independently without user context, use client credentials

2. **Token Management Strategy**
   - **Token Exchange**: Use OAuth 2.0 token exchange (RFC 8693) to convert user tokens into agent-scoped tokens with proper actor claims
   - **Actor Claims**: Include `act` claim in tokens to identify which agent is performing actions, creating an audit trail
   - **Token Lifecycle**: Implement short-lived access tokens with refresh mechanisms for long-running operations

3. **Delegation Patterns**
   - **Direct Agent Authentication**: Agents with dedicated OAuth clients authenticate directly
   - **Mediator Pattern**: Multiple agents share a common OAuth client through a mediator, requiring actor tokens for individual agent identification
   - **Multi-Agent Chains**: Support nested delegation where agents delegate to other agents, maintaining the full chain in token claims

4. **Consent and Authorization Flows**
   - **Pre-authorized Consent**: Check for existing user consent before proceeding with sensitive operations
   - **Dynamic Consent**: Request additional consent when needed using OIDC authorization flows or CIBA
   - **Consent Granularity**: Allow users to consent to specific agents for specific purposes with specific scopes

5. **Standards-Based Approach**: A2A leverages established web security practices:
   - **OAuth 2.0 Flows**: Client credentials, token exchange, CIBA, authorization code
   - **Rich Authorization Requests**: Fine-grained permissions beyond simple scopes
   - **Transaction Tokens**: Purpose-specific tokens for single operations

6. **Implementation Considerations**
   - **Authorization Server Capabilities**: Ensure support for token exchange, actor validation, and agent-specific policies
   - **Credential Security**: Secure storage, rotation, and revocation of agent credentials
   - **Monitoring and Auditing**: Comprehensive logging of authorization decisions and agent actions
   - **Error Handling**: Graceful handling of insufficient permissions with clear feedback for consent requests

## Diving into authorization for different Agent Types

Broadly, agents can be categorized into three types based on how they are invoked and the context in which they perform tasks:

1. **Conversational Agents**: Invoked by user prompts, have access to user tokens, need to exchange for appropriate permissions
2. **Autonomous Agents (User Context)**: Run in background on behalf of users, may need CIBA or refresh tokens for consent
3. **Autonomous Agents (System Context)**: Operate independently without user context, use client credentials

Each type of Agent requires different authorization mechanisms to obtain permissions to perform tasks. The following sections dive deeper into this topic.

### Conversational Agent

Conversational Agents are designed to interact with humans through natural language processing (NLP) and act based on instructions. They are typically used to provide customer service, answer questions, or perform tasks that may require calling into other Agents.

Characteristics of a conversational agent -

- Invoked through a user prompt
- Agent would have access to a security token, like a OAuth 2.0 access token, that can be used to assert the identity of the user
- The user token will likely not have the permissions needed to perform tasks or rights to call the A2A server

The typical flow and steps to obtain an appropriate authorization token is illustrated below. Broadly, the flow would include the following steps:

1. User authenticates and authorizes access to the application that presents the user interface
2. User initiates a task by issuing a prompt, typically
3. The application invokes a supervisor agent with the user's token
     - The user tokens should be low privileged and may not authorize access to the APIs that the Agents need to complete the task
     - The user's token should be [sender-constrained](https://docs.verify.ibm.com/verify/docs/oauth-20-dpop) to prevent the Agent from directly using it to call authorized APIs
4. The supervisor agent obtains the requirements to call into a supporting agent or agents and presents it with a on-behalf-of (OBO) token that embeds both the user and the supervisor agent identity
     - Uses [OAuth 2.0 Token Exchange](https://docs.verify.ibm.com/verify/docs/oauth-20-token-exchange)
5. The supporting agent validates the token and either performs the task directly or, in the event that it needs to call into an external system, obtains appropriate tokens or credentials
     - For endpoints authorized with tokens issued by the same OAuth authorization server: The supporting agent exchanges the OBO token for a new token that contains the necessary permissions, the identity of the user and the identity of both the supervisor and supporting agent
     - For endpoints authorized with tokens issued by external OAuth authorization server: The supporting agent exchanges the OBO token for a new token from the external OAuth authorization server, assuming it supports token exchange
     - For endpoints authorized with credentials/tokens that are not OAuth based: The supporting agent presents the OBO token to a system like Hashicorp Vault to obtain a static or dynamic credential that permits it to connect to the endpoint(s)

The sequence diagram below offers a more exhaustive view of the steps.

**Scenario: User wants to check if her calendar is open**

<pre class="mermaid">
sequenceDiagram
    autonumber
    participant Jessica as User Jessica
    participant AgenticApp as Agentic Application
    participant AuthServer as IBM Verify<br/>(Authorization Server)
    participant AIAgent as A2A Client<br/>(Supervisor Agent)
    participant A2AServer as A2A Server<br/>(Supporting Agent)
    participant APIs as APIs<br/>(Email/Calendar/Docs)
    
    Note over Jessica,AuthServer: Initial Authentication & Authorization
    Jessica->>AgenticApp: Access application
    AgenticApp->>AuthServer: OIDC Authorization Request
    AuthServer->>Jessica: Authentication prompt
    Jessica->>AuthServer: Authenticate & grant consent
    AuthServer->>AgenticApp: Authorization code
    AgenticApp->>AuthServer: Exchange code for token
    AuthServer->>AgenticApp: Access token (limited privileges)
    
    Note over AgenticApp,AIAgent: AI Agent Invocation
    AgenticApp->>AIAgent: Is the user available on Thursday at 10 AM?
    
    Note over AIAgent,AuthServer: Obtain the A2A client scoped access token<br/>that contains the user and agent identity
    AIAgent->>AuthServer: POST /token<br/>grant_type=token-exchange<br/>subject_token=<Jessica's token><br/>actor_token=<A2A client token><br/>
    AuthServer->>AuthServer: Validate if the A2A Client can act on behalf of the user
    AuthServer->>AuthServer: Audit the token issuance with the actor information
    AuthServer-->>AIAgent: Delegated access token<br/>(with "act" claim that identifies the AI Agent)

    Note over AIAgent,A2AServer: Execute the task
    AIAgent->>A2AServer: Get A2A Agent Card
    A2AServer-->>AIAgent: A2A Agent Card
    AIAgent->>A2AServer: Check if the user's calendar is open for Thursday at 10 AM<br/>{delegated_access_token}
    A2AServer->>AuthServer: POST /token<br/>grant_type=token-exchange<br/>subject_token=<delegated_access_token><br/>actor_token=<A2A server token><br/>scope=calendar:read
    AuthServer->>AuthServer: Validate the A2A Server can act on behalf of the user
    AuthServer->>AuthServer: Check for user consent
    
    alt User has consented to share calendar

    AuthServer->>AuthServer: Audit the token issuance with the hierarchy of actors:<br/>A2A Server and A2A Client
    AuthServer-->>A2AServer: Delegated access token<br/>(with "act" claim that identifies the AI Agent)
    A2AServer->>APIs: Get calendar data
    A2AServer->>A2AServer: Compute availability
    A2AServer-->>AIAgent: User is available
    AIAgent-->>AgenticApp: User is available

    else User has not consented

    AuthServer-->>A2AServer: insufficient_scope error
    A2AServer-->>AIAgent: insufficient_scope error
    AIAgent-->>AgenticApp: Consent requested for scope=calendar:read
    AgenticApp->>AuthServer: OIDC authorization request with scope=calendar:read
    AuthServer->>Jessica: Ask for consent
    Jessica->>AuthServer: Consent granted
    AuthServer->>AuthServer: Audit the consent, including who this is meant for,<br/>which includes the requesting actor (A2A Server)
    AuthServer->>AgenticApp: New access token with requested privileges<br/>Flow is repeated at this point to invoke the agent
    
    end
</pre>

Notice that given access to the user interface in this scenario, the flow leverages OIDC authorization code flow to request for additional consent. This can optionally be replaced with OAuth 2.0 Client Initiated Backchannel Authentication. However, the user would still need to be informed through the chat-type interface that a notification has been sent to their device for approval.

### Autonomous Agent acting on behalf of a user

Autonomous Agents run in the background to perform tasks. They can act on behalf of a user in certain cases.

Characteristics of such an autonomous agent -

- Invoked through some form of a system event with no human intervention
- Agent needs to perform tasks on behalf of a user
- Agent may or may not have a long-lived token available representing the user

The typical flow and steps to obtain an appropriate authorization token is illustrated below. Broadly, the flow would include the following steps:

1. The supervisor agent obtains the requirements to call into a supporting agent or agents and presents it with a on-behalf-of (OBO) token that embeds both the user and the supervisor agent identity
     - Uses [OAuth 2.0 Token Exchange](https://docs.verify.ibm.com/verify/docs/oauth-20-token-exchange) if the supervisor agent has access to a long-lived user token, such as an OAuth 2.0 refresh token
     - Uses [OAuth 2.0 Client-Initiated Backchannel Authentication (CIBA)](https://docs.verify.ibm.com/ibm-security-verify-access/docs/oauth2-ciba_overview) if the supervisor agent does not have a user token available. Through this process, the user completes authentication and approves for any solicited consent
     - In both cases, the supervisor agent's identity token should be provided to the request to audit the actor acting on behalf of the user
2. The supporting agent validates the token and either performs the task directly or, in the event that it needs to call into an external system, obtains appropriate tokens or credentials
     - For endpoints authorized with tokens issued by the same OAuth authorization server: The supporting agent exchanges the OBO token for a new token that contains the necessary permissions, the identity of the user and the identity of both the supervisor and supporting agent
     - For endpoints authorized with tokens issued by external OAuth authorization server: The supporting agent exchanges the OBO token for a new token from the external OAuth authorization server, assuming it supports token exchange
     - For endpoints authorized with credentials/tokens that are not OAuth based: The supporting agent presents the OBO token to a system like Hashicorp Vault to obtain a static or dynamic credential that permits it to connect to the endpoint(s)

The sequence diagram below offers a more exhaustive view of the steps.

**Scenario: Autonomous Scheduling Agent identifying available slots for a periodic All Hands meeting**

<pre class="mermaid">
sequenceDiagram
    autonumber
    participant Jessica as User Jessica
    participant AuthServer as IBM Verify<br/>(Authorization Server)
    participant AIAgent as A2A Client<br/>(Scheduling Agent)
    participant A2AServer as A2A Server<br/>(Calendar Agent)
    participant APIs as APIs<br/>(Email/Calendar/Docs)
    
    AIAgent->>A2AServer: Get A2A Agent Card to determine scopes required
    A2AServer-->>AIAgent: A2A Agent Card
    
    Note over AIAgent,AuthServer: Obtain the A2A client scoped access token<br/>that contains the user and agent identity

    alt Long-lived refresh token is not available

    AIAgent->>AuthServer: Request user token using<br/>Client-initiated backchannel authentication (CIBA)<br/>requested_actor={A2A Client token}<br/>scope=calendar:read
    AuthServer->>Jessica: Request for authentication and consent<br/>(through push or other registered mechanisms)
    Jessica->>AuthServer: Authenticates and approves
    AuthServer->>AuthServer: Audit the token issuance with the actor information
    AuthServer-->>AIAgent: User token with actor information<br/>{delegated_user_token}

    else

    AIAgent->>AuthServer: POST /token<br/>grant_type=token-exchange<br/>subject_token=<refresh_token><br/>actor_token=<agent_token>
    AuthServer->>AuthServer: Audit the token issuance with the actor information
    AuthServer-->>AIAgent: User token with actor information<br/>{delegated_user_token}

    end

    Note over AIAgent,A2AServer: Execute the task
    AIAgent->>A2AServer: Look for free slots in the user's calendar 3 weeks from now<br/>{delegated_access_token}
    A2AServer->>AuthServer: POST /token<br/>grant_type=token-exchange<br/>subject_token=<delegated_access_token><br/>actor_token=<A2A server token><br/>scope=calendar:read
    AuthServer->>AuthServer: Validate the A2A Server can act on behalf of the user
    AuthServer->>AuthServer: Check for user consent
    
    alt User has consented to share calendar

    AuthServer->>AuthServer: Audit the token issuance with the hierarchy of actors:<br/>A2A Server and A2A Client
    AuthServer-->>A2AServer: Delegated access token<br/>(with "act" claim that identifies the AI Agent)
    
    else User has not consented

    AuthServer-->>A2AServer: insufficient_scope error
    A2AServer-->>AIAgent: insufficient_scope error
    AIAgent->>AuthServer: Request user token using<br/>Client-initiated backchannel authentication (CIBA)<br/>requested_actor={A2A Server token}<br/>scopes=calendar:read
    AuthServer->>Jessica: Request for authentication and consent<br/>(through push or other registered mechanisms)
    Jessica->>AuthServer: Authenticates and approves
    AuthServer->>AuthServer: Audit the token issuance with the actor information
    AuthServer-->>AIAgent: User token with actor information<br/>{delegated_user_token}

    end

    A2AServer->>APIs: Get calendar data
    A2AServer->>A2AServer: Compute availability
    A2AServer-->>AIAgent: Return available slots
</pre>

### Autonomous Agent acting without human intervention

Autonomous Agents run in the background to perform tasks. Here, the agent performs tasks without any human intervention or context.

Characteristics of such an autonomous agent -

- Invoked through some form of a system event with no human intervention
- Agent needs to perform tasks without user context or consent

The typical flow and steps to obtain an appropriate authorization token is illustrated below. Broadly, the flow would include the following steps:

1. The supervisor agent obtains the requirements to call into a supporting agent or agents and presents it with a token that has the necessary permissions
     - Uses OAuth 2.0 Client Credentials Grant Flow to obtain the token
2. The supporting agent validates the token and performs the task using the token
     - In some cases, the supporting agent may need to exchange the token for a delegated system token with additional permissions. This uses OAuth 2.0 Token Exchange flow.

The sequence diagram below offers a more exhaustive view of the steps.

**Scenario: Check for support tickets**

<pre class="mermaid">
sequenceDiagram
    autonumber
    participant AuthServer as IBM Verify<br/>(Authorization Server)
    participant AIAgent as A2A Client<br/>(Supervisor Agent)
    participant A2AServer as A2A Server<br/>(Support Agent)
    participant APIs as APIs<br/>(Tickets)
    
    AIAgent->>A2AServer: Get A2A Agent Card to determine scopes required
    A2AServer-->>AIAgent: A2A Agent Card
    
    Note over AIAgent,AuthServer: Obtain a token with sufficient permissions
    AIAgent->>AuthServer: POST /token<br/>grant_type=client_credentials<br/>client_id=<agent_client_id><br/>client_secret=<agent_client_secret><br/>scope=service_tickets
    AuthServer-->>AIAgent: Access token with sufficient privileges

    Note over AIAgent,A2AServer: Perform the task
    AIAgent->>A2AServer: List open support tickets<br/>{access_token}
    A2AServer->>APIs: Get tickets
    APIs-->>A2AServer: List of open support tickets
    A2AServer-->>AIAgent: List of open support tickets
</pre>

An extension to this flow may use the `{access_token}` to exchange for another token at the A2A server. This is relevant for cases where the A2A Client may obtain a more generic token that embeds transaction context. This transaction token is exchanged for the actual token that can be used to call APIs.

## Advanced concepts

### Representing authorization requests

While the examples here used coarse-grained OAuth 2.0 scopes, in many cases, it is important to represent permissions as fine-grained authorization objects that constrain what is permitted and allows for context-based access control on the Authorization Server prior to issuing tokens. In this scenario, use [OAuth 2.0 Rich Authorization Request](https://datatracker.ietf.org/doc/html/rfc9396). 

Consider the case where the Agent needed the permission to read the calendar. It obtained this permission by requesting for the `calendar:read` scope. The scope doesn't indicate the specific calendar and entry types that are permitted to be read. With this, the Agent may receive a wider range of permissions than desired.

Now, consider how this might be represented using an `authorization_details` object:

```json
{
    "type": "calendar_access",
    "actions": [
        "read"
    ],
    "datatypes": [
        "events",
        "appointments"
    ],
    "calendar_ids": [
        "work"
    ]
}
```

Notice here that the specific `datatypes` and `calendar_ids` restrict access in a more fine-grained manner. In the spirit of least privileged access, this is the recommended approach, unless an Agent truly needs a wide range of access.

### Multi-Agent flows and transaction tokens

[OAuth 2.0 Transaction Tokens](https://tools.ietf.org/html/draft-ietf-oauth-transaction-tokens-01) introduce a method of representing the transaction context as an immutable part of a short-lived Transaction Token (txn-token). This establishes the intent or goal of the flow.

With this, in a multi-agent flow in particular, Agents can directly use this txn-token in a variety of ways:

- When accessing endpoints secured with tokens issued by the same OAuth 2.0 authorization server, exchange the txn-token for an appropriate bearer token. The transaction context can be used for context-aware decisions.
- When accessing endpoints secured with tokens issued by a different authorization server, exchange the txn-token for an appropriate bearer token.
- When accessing endpoints secured with non-OAuth credentials or tokens that are statically stored in, say, Hashicorp Vault or dynamically generated, use the txn-token to authenticate the request to the Vault.
- When performing tasks without needing to access endpoints, use the txn-token to obtain transaction context information and act based on that.

This is still an evolving standard.

## Summary

Authorization is fundamental to building secure, trustworthy, and user-centric Agent-to-Agent systems. As AI agents increasingly collaborate through protocols like A2A, proper authorization ensures:

- **User privacy and control** through explicit consent mechanisms and fine-grained permissions
- **Security and accountability** via comprehensive audit trails tracking both user and agent identities
- **Flexible delegation** supporting complex multi-agent workflows while maintaining trust chains
- **Standards-based implementation** leveraging proven OAuth 2.0 patterns and extensions

While the A2A protocol leaves authorization to the implementer, this article establishes prescriptive patterns to secure and authorize multi-agent flows using IBM Verify as the authorization server. The key takeaways are:

1. **Match the OAuth flow to your agent type:**
   - **Conversational agents**: Use token exchange with user tokens to maintain user context
   - **Autonomous agents (user context)**: Leverage CIBA or refresh tokens for asynchronous consent
   - **Autonomous agents (system context)**: Use client credentials for machine-to-machine flows

2. **Implement proper delegation chains:**
   - Use actor tokens to identify agents in authorization decisions
   - Maintain nested `act` claims for multi-agent scenarios
   - Enable authorization servers to apply agent-specific policies

3. **Prioritize user consent and transparency:**
   - Check for pre-authorized consent before sensitive operations
   - Request dynamic consent when needed with clear agent identification
   - Provide users with visibility and control over agent permissions

4. **Adopt advanced patterns when appropriate:**
   - Use Rich Authorization Requests for fine-grained permissions beyond simple scopes
   - Consider transaction tokens for immutable transaction context in complex flows
   - Implement sender-constrained tokens to prevent token misuse

By following these patterns, organizations can build A2A systems that balance the power and autonomy of AI agents with the security, privacy, and control that users and enterprises require. The combination of established OAuth 2.0 standards with agent-specific extensions provides a robust foundation for the next generation of multi-agent AI systems.

<script src="https://cdn.jsdelivr.net"></script>
<script>mermaid.initialize({startOnLoad:true});</script>