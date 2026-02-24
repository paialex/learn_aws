Act as an Expert AWS Solutions Architect. Your task is to author a comprehensive, highly visual guide on [AWS API Gateway].

**Execution Requirements (MCP):**
1. Use your file system MCP tool to create a dedicated folder named `{service}-visual-guide`.
2. Inside this folder, create a Markdown document named `architecture-patterns.md` containing the article.

**Content Requirements:**
Structure the article with the following logical sections:
1. **Core Functionalities:** Explain primary capabilities (e.g., Request Routing, Authentication/Authorization, Throttling, Protocol Translation, Payload Transformation).
2. **Use Cases:** Define when to use REST APIs vs. HTTP APIs vs. WebSocket APIs.
3. **Architectural Patterns:** Detail common patterns such as Centralized Edge Gateway, Backend-for-Frontend (BFF), Microgateway, and Strangler Fig.
4. **Architectural Anti-Patterns:** Detail anti-patterns such as the Monolithic Gateway (Single Point of Failure), embedding deep business logic in the Gateway, and lack of timeout/retry configurations.

**Visualization Requirements (CRITICAL):**
I process information visually. For *every single* pattern and anti-pattern discussed, you MUST include a comprehensive diagram. The diagrams must map the explicit traffic flow between Clients, the API Gateway, Security layers (e.g., Cognito/Lambda Authorizers), and Backend services (Lambda, ECS, VPC Links, etc.). Do not omit the visual diagram for any explained architecture.
