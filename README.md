# Auspost Salesforce Integration Demo Application

This is a demonstration MuleSoft application designed to integrate with Salesforce. It acts as a proxy, abstracting the direct Salesforce authentication and authorization flows. This setup enables organizations like Auspost to adhere to internal best practices by leveraging their existing Identity Provider (IdP), such as Microsoft Entra ID (Azure AD), for client authentication.

## Purpose

The primary goal of this application is to solve a specific challenge:

*   **Compliance with Internal Best Practices:** Auspost requires all applications to use their corporate IdP (Azure AD/Entra ID) for authentication. The standard Salesforce authentication flows don't natively support this, creating a compliance gap.
*   **Centralized Authentication:** By proxying requests through this MuleSoft application, Auspost can enforce a single source of truth for authentication using Entra ID.
*   **Extended Capabilities:** Utilizing Mulesoft allows the addition of extra capabilities such as logging, tracing, and custom policies, which can be managed in Anypoint Platform.

## Architecture Overview

The application functions as follows:

1.  **Client Request:** An external client (e.g., a web application) makes a request to the MuleSoft application endpoint. This endpoint is secured by Azure AD using Mulesoft API Manager.
2.  **Entra ID Authentication:** The client authenticates against Azure AD, obtaining a token.
3.  **MuleSoft API Manager Validation:** The API Manager validates the token issued by Entra ID, ensuring the client is authorized to access the API.
4.  **Salesforce Integration:** The MuleSoft application, upon successful authentication and authorization, interacts with Salesforce on behalf of the client.
5.  **Data Transformation and Routing:** The MuleSoft application can transform data between the client and Salesforce as needed and routes the request to the appropriate Salesforce endpoint.
6.  **Response to Client:** The response from Salesforce is processed (if necessary) and returned to the client.

## Getting Started

Before you can deploy and run this application, you'll need the following:

1.  **Anypoint Platform Account:** A MuleSoft Anypoint Platform account to deploy and manage the application.
2.  **Salesforce Account:** Access to a Salesforce instance with appropriate permissions.
3.  **Microsoft Entra ID (Azure AD) Tenant:** An Azure AD tenant configured with application registrations for client authentication and authorization.
4.  **MuleSoft API Manager Configuration:** API Manager configured to validate JWT tokens issued by Entra ID.

### Salesforce Configuration

The application uses a JWT (JSON Web Token) Bearer Flow to authenticate with Salesforce.  This requires the following Salesforce configuration:

1.  **Connected App:** Create a Connected App in Salesforce.
    *   Enable OAuth Settings.
    *   Select "Use digital signatures" to use a JWT.
    *   Upload a digital certificate.
    *   Define the OAuth scopes required for your integration.
2.  **User Account:** Ensure there's a Salesforce User that the connected app can access, ensure you have their email/username as you need to add it to "Permitted Users"

### Application Configuration

The core configuration for Salesforce connectivity is found in `src/main/mule/auspost-salesforce.xml`.

**Important Security Note:** The following credentials are used for demonstration purposes ONLY and MUST be replaced with your own.

*   **`consumerKey` (Salesforce Connected App Client ID):** Located in the `salesforce:jwt-connection` element within the `Salesforce_Config` configuration. This should be the Consumer Key (Client ID) from your Salesforce Connected App.
*   **`keyStore` (Path to the JKS Keystore):**  This is the path to the JKS keystore file containing the private key used to sign the JWT.  Example: `keystore.jks`. The path is in /src/main/resources/, when reading this path, you can just use the filename.
*   **`storePassword` (Keystore Password):**  The password for the keystore file. Example: `yourpass`
*   **`principal` (Salesforce Username):** This value is being dynamically set in each flow.

**To configure Salesforce connectivity:**

1.  **Replace Dummy Credentials:** Replace the placeholder values in the `Salesforce_Config` with your actual Salesforce Connected App credentials.
2.  **Configure JWT Keystore:** Ensure the correct keystore file (`keystore.jks`) is in the appropriate location and that the `storePassword` is correct. The keystore needs to contain the private key that matches the certificate uploaded to your Salesforce Connected App.
3.  **Principal Salesforce User:** The variable 'principal' is being dynamically set in each flow. Set the appropriate Salesforce username (e.g., an integration user) to the variable to perform actions in salesforce using the integration user's permission sets. This injects the variable at runtime via the header "FederationId"

### Entra ID (Azure AD) Configuration

1.  **Application Registration:** Register an application in Azure AD.
2.  **API Permissions:** Configure API permissions for your Azure AD application to allow access to the MuleSoft application endpoint.  This typically involves defining scopes and granting consent.
3.  **Client Credentials:** Obtain the Client ID and Client Secret for your Azure AD application. These are used by clients to authenticate against Azure AD and obtain tokens.

### Mulesoft Anypoint API Manager Configuration

1.  **API Definition:** Import or define your API in Anypoint API Manager.
2.  **Apply Policy:** Apply a "JWT Validation" policy to your API. Configure the policy to validate JWT tokens issued by Azure AD, including the correct issuer and audience. You might also set the policy to extract information from JWT for use within the mule application.
3.  **Associate API to Application:** Create an application within Anypoint Exchange and associate it with the API. This will generate a client ID and secret (or JWT) that clients use to access the API.

## Flows

The application contains the following main flows:

*   **`auspost-salesforce-main`:** The main flow handles incoming HTTP requests and routes them to the appropriate APIKit router.
*   **`auspost-salesforce-console`:** Provides an APIKit console for testing the API.
*   **`get:\sf-read:auspost-salesforce-config`:** Retrieves account records from Salesforce.
*   **`post:\sf-create:application\json:auspost-salesforce-config`:** Creates a new Account in Salesforce, taking request body (JSON payload).
     * FederationID header is picked from the request and used as the principal user, so each service principal can act as different Salesforce user.

## API Endpoints

The following API endpoints are available for interacting with the application. All endpoints require a valid Bearer Token obtained from Azure AD.

### `GET /api/sf-read`

*   **Description:** Retrieves account records from Salesforce.
*   **Request:**
    *   **Method:** `GET`
    *   **Headers:**
        *   `Authorization: Bearer <YOUR_AZURE_AD_TOKEN>`
*   **Response:**
    *   **Status Code:** `200 OK` (on success)
    *   **Body:** A JSON array containing Salesforce account records. Example:

        ```json
        [
          {
            "Id": "001xxxxxxxxxxxxxxx",
            "Name": "Acme Corp",
            "AccountNumber": "12345"
          },
          {
            "Id": "001yyyyyyyyyyyyyyy",
            "Name": "Beta Inc",
            "AccountNumber": "67890"
          }
        ]
        ```

### `POST /api/sf-create`

*   **Description:** Creates a new Account in Salesforce.
*   **Request:**
    *   **Method:** `POST`
    *   **Headers:**
        *   `Authorization: Bearer <YOUR_AZURE_AD_TOKEN>`
        *   `Content-Type: application/json`
        *   `FederationId`: Salesforce Username that should be used
    *   **Body:** A JSON array containing the account data to create. Example:

        ```json
        [
          {
            "Name": "Amazon Corp",
            "AccountNumber": "123"
          }
        ]
        ```
*   **Response:**
    *   **Status Code:** `201 Created` (on success)
    *   **Body:** A JSON array containing the newly created Salesforce account record IDs. Example:

        ```json
        [
          {
            "id": "001zzzzzzzzzzzzzzz"
          }
        ]
        ```

## Obtaining an Azure AD Bearer Token

To access the API endpoints, you must first obtain a Bearer Token from Azure AD.  You can do this by making a POST request to the Microsoft Online endpoint:

```
https://login.microsoftonline.com/{your-tenant-id}/oauth2/v2.0/token
```

Replace `{your-tenant-id}` with your Azure AD tenant ID.

**Request Body (x-www-form-urlencoded):**

*   `grant_type`: `client_credentials`
*   `client_id`: `<YOUR_AZURE_AD_CLIENT_ID>`
*   `client_secret`: `<YOUR_AZURE_AD_CLIENT_SECRET>`
*   `scope`: `<YOUR_API_SCOPE>` (The scope defined when registering the application in Entra ID. This should match what you configure in Anypoint API Manager.)

**Example using `curl`:**

```bash
curl -X POST \
  'https://login.microsoftonline.com/<YOUR_AZURE_TENANT_ID>/oauth2/v2.0/token' \
  -H 'Content-Type: application/x-www-form-urlencoded' \
  -d 'grant_type=client_credentials&client_id=<YOUR_AZURE_AD_CLIENT_ID>&client_secret=<YOUR_AZURE_AD_CLIENT_SECRET>&scope=<YOUR_API_SCOPE>'
```

The response will contain an `access_token` field, which is your Bearer Token. Use this token in the `Authorization` header of your API requests.

## Areas for Improvement - Production Readiness

This demonstration application provides a basic framework for integrating Salesforce with Azure AD using MuleSoft. To make it production-ready, several areas require enhancement:

*   **Comprehensive Logging:** Implement detailed logging to track requests, responses, errors, and other relevant events. Use a structured logging format and consider integrating with a central logging system for analysis and monitoring.  Consider using the Log4j2 framework within MuleSoft.
*   **Robust Error Handling:** Implement a more sophisticated error handling strategy that includes:
    *   Specific exception handling for different error scenarios (e.g., Salesforce API errors, network errors, authentication errors).
    *   Retry mechanisms for transient errors.
    *   Circuit breaker pattern to prevent cascading failures.
    *   Centralized error reporting and alerting.
*   **Monitoring and Alerting:** Integrate the application with a monitoring solution (e.g., Prometheus, CloudWatch) to track performance metrics, error rates, and other key indicators.  Set up alerts to notify administrators of critical issues.
*   **Security Hardening:** Review and implement security best practices, including:
    *   Input validation to prevent injection attacks.
    *   Secure storage of sensitive data (e.g., using the MuleSecure Properties tool).
    *   Regular security audits and penetration testing.
*   **API Versioning:** Implement API versioning to allow for future changes without breaking existing clients.
*   **Throttling and Rate Limiting:** Implement throttling and rate limiting to protect the Salesforce instance and the MuleSoft application from overload. This can be configured within API Manager.
*   **Unit and Integration Testing:** Write comprehensive unit and integration tests to ensure the application's functionality and stability.
*   **Configuration Management:** Externalize configuration parameters (e.g., Salesforce credentials, API endpoints) and manage them using a configuration management tool.
*   **Documentation:** Create thorough documentation for the application, including API documentation, deployment instructions, and troubleshooting guides.

## Error Handling

The application includes error handling for common APIKit errors, such as:

*   `APIKIT:BAD_REQUEST`
*   `APIKIT:NOT_FOUND`
*   `APIKIT:METHOD_NOT_ALLOWED`
*   `APIKIT:NOT_ACCEPTABLE`
*   `APIKIT:UNSUPPORTED_MEDIA_TYPE`
*   `APIKIT:NOT_IMPLEMENTED`

These errors are caught and transformed into JSON responses with appropriate HTTP status codes.
Explicit error handling beyond APIKIT errors is not implemented.

## Next Steps

*   **Deploy the Application:** Deploy the MuleSoft application to your Anypoint Platform environment.
*   **Configure API Manager:** Configure API Manager to secure the API with Azure AD authentication.
*   **Test the Integration:** Test the integration by sending requests to the API endpoints and verifying the data in Salesforce.
*   **Customize and Extend:** Customize and extend the application to meet your specific integration requirements.

This demonstration provides a foundation for building a secure and compliant Salesforce integration using Azure AD and MuleSoft. Remember to always prioritize security and replace the example credentials with your own before deploying the application to a production environment.
