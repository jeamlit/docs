---
title: Use Microsoft Entra to authenticate users
slug: /develop/tutorials/authentication/microsoft
description: Learn how to authenticate users with Microsoft Entra
ignore: false
---

# Use Microsoft Entra to authenticate users

[Microsoft Identity Platform](https://learn.microsoft.com/en-us/entra/identity-platform/v2-overview) is a service within Microsoft Entra that lets you build applications to authenticate users. Your applications can use personal, work, and school accounts managed by Microsoft.

## Prerequisites

- This tutorial requires the following Java libraries:

  ```text
  //DEPS com.google.code.gson:gson:2.10.1
  //DEPS com.nimbusds:nimbus-jose-jwt:10.7
  //DEPS org.apache.httpcomponents.client5:httpclient5:5.3.1
  ```

- You should have a clean working directory called `your-repository`.
- You must have a Microsoft Azure account, which includes Microsoft Entra ID.

## Summary

In this tutorial, you'll build an app that users can log in to with their personal Microsoft accounts. When they log in, they'll see a personalized greeting with their name and have the option to log out.

Here's a look at what you'll build:

<Collapse title="Complete code" expanded={false}>

`.env`

```bash
ENTRA_CLIENT_ID=xxx
ENTRA_CLIENT_SECRET=xxx
ENTRA_TENANT_ID=xxx
ENTRA_REDIRECT_URL=http://localhost:8501/auth/callback
```

`AppWithEntra.java`

```java
//DEPS com.google.code.gson:gson:2.10.1
//DEPS com.nimbusds:nimbus-jose-jwt:10.7
//DEPS org.apache.httpcomponents.client5:httpclient5:5.3.1


import java.io.UnsupportedEncodingException;
import java.net.URLEncoder;
import java.nio.charset.StandardCharsets;

import com.google.gson.JsonObject;
import com.google.gson.JsonParser;
import com.nimbusds.jwt.SignedJWT;
import com.nimbusds.jwt.JWTClaimsSet;


import java.util.*;

import org.apache.http.HttpEntity;
import org.apache.http.client.methods.HttpPost;
import org.apache.http.entity.StringEntity;
import org.apache.http.impl.client.CloseableHttpClient;
import org.apache.http.impl.client.HttpClients;
import org.apache.http.util.EntityUtils;

import io.javelit.core.Jt;

public class AppWithEntra {

    private static final String SESSION_USER = "current_user";
    private static final String SESSION_TOKEN = "auth_token";
    private static final String clientId = System.getenv("ENTRA_CLIENT_ID");
    private static final String clientSecret = System.getenv("ENTRA_CLIENT_SECRET");
    private static final String tenantId = System.getenv("ENTRA_TENANT_ID");
    private static final String redirectUri = System.getenv("ENTRA_REDIRECT_URL");
    private static final String tokenEndpoint = String.format(
                "https://login.microsoftonline.com/%s/oauth2/v2.0/token", tenantId);
    private static final String authorizationEndpoint = String.format(
                "https://login.microsoftonline.com/%s/oauth2/v2.0/authorize", tenantId);

    public static void main(String[] args) throws UnsupportedEncodingException {
        Jt.title("Welcome to Javelit! \uD83D\uDEA1").use();
        
        boolean loggedIn = Jt.sessionState().computeIfAbsentBoolean("logged_in", k -> false);

        if (!loggedIn) {
            var currentPage = Jt.navigation(Jt.page("/login", () -> {
                loginPage();
            }), Jt.page("/auth/callback", () -> {
                renderCallbackPage();
                Jt.rerun(true);
            })).hidden().use();
            currentPage.run();
        } else {
            var currentPage = Jt.navigation(Jt.page("/dashboard", () -> {
                renderDashboardPage();
            }), Jt.page("/logout", () -> {
                Jt.sessionState().clear();
                Jt.rerun(true);
            }).title("Logout")).use();
            currentPage.run();
        }
    }

    private static void loginPage() throws UnsupportedEncodingException {
        Jt.text("Please authenticate with your Microsoft 365 account").use();

        // Generate a state parameter for CSRF protection
        String state = UUID.randomUUID().toString();

        // Get the authorization URL
        String authUrl = getAuthorizationUrl(state);

        Jt.pageLink(authUrl, "Login with Microsoft").target("_self").use();
    }

    private static void renderDashboardPage() {
        Jt.header("Dashboard").use();
        UserClaims userClaims = (UserClaims) Jt.sessionState().get(SESSION_USER);
        Jt.markdown("You are logged in: " + userClaims.getName()).use();
        userClaims.getRoles().forEach(role -> {
            Jt.markdown("Role: " + role).use();
        });
    }

    public static String getAuthorizationUrl(String state) throws UnsupportedEncodingException {
        Map<String, String> params = new LinkedHashMap<>();
        params.put("client_id", clientId);
        params.put("response_type", "code");
        params.put("redirect_uri", redirectUri);
        params.put("scope", "openid profile email Directory.Read.All");
        params.put("state", state);

        return authorizationEndpoint + "?" + buildQueryString(params);
    }

    private static String buildQueryString(Map<String, String> params) throws UnsupportedEncodingException {
        StringBuilder sb = new StringBuilder();
        for (Map.Entry<String, String> entry : params.entrySet()) {
            if (sb.length() > 0) {
                sb.append("&");
            }
            sb.append(URLEncoder.encode(entry.getKey(), "UTF-8"));
            sb.append("=");
            sb.append(URLEncoder.encode(entry.getValue(), "UTF-8"));
        }
        return sb.toString();
    }

    private static void renderCallbackPage() {

        Map<String, List<String>> params = Jt.urlQueryParameters();

        // Get the authorization code and state from the query parameters
        String code = Optional.ofNullable(params.get("code")).map(l -> l.get(0)).orElse(null);
        String state = Optional.ofNullable(params.get("state")).map(l -> l.get(0)).orElse(null);
        List<String> error = Optional.ofNullable(params.get("error")).orElse(List.of());
        List<String> errorDescription = Optional.ofNullable(params.get("error_description")).orElse(List.of());
        
        // Check for errors from Microsoft
        if (!error.isEmpty()) {
            renderErrorPage(error.get(0),
                    !errorDescription.isEmpty() ? errorDescription.get(0) : "");
            return;
        }

        try {
            // Process the authorization code and get tokens
            TokenResponse tokenResponse = getTokenFromAuthCode(code);
            UserClaims userClaims = parseToken(tokenResponse.getIdToken());
            // Store user info and tokens in session
            Jt.sessionState().put(SESSION_USER, userClaims);
            Jt.sessionState().put(SESSION_TOKEN, tokenResponse.getAccessToken());
            Jt.sessionState().put("logged_in", true);
        } catch (SecurityException e) {
            // State validation failed - potential CSRF attack
            renderErrorPage("Security Error", "Invalid state parameter - possible CSRF attack");
        } catch (Exception e) {
            // Token exchange failed
            renderErrorPage("Authentication Failed", e.getMessage());
        }
    }

    public static TokenResponse getTokenFromAuthCode(String authCode) throws Exception {
        try (CloseableHttpClient httpClient = HttpClients.createDefault()) {
            HttpPost httpPost = new HttpPost(tokenEndpoint);

            Map<String, String> params = new LinkedHashMap<>();
            params.put("client_id", clientId);
            params.put("client_secret", clientSecret);
            params.put("code", authCode);
            params.put("redirect_uri", redirectUri);
            params.put("grant_type", "authorization_code");
            params.put("scope", "openid profile email");

            httpPost.setEntity(new StringEntity(buildQueryString(params), StandardCharsets.UTF_8));
            httpPost.setHeader("Content-Type", "application/x-www-form-urlencoded");

            return httpClient.execute(httpPost, response -> {
                HttpEntity entity = response.getEntity();
                String responseBody = EntityUtils.toString(entity);
                return parseTokenResponse(responseBody);
            });
        }
    }

    private static TokenResponse parseTokenResponse(String responseBody) {
        JsonObject jsonObject = JsonParser.parseString(responseBody).getAsJsonObject();

        TokenResponse tokenResponse = new TokenResponse();
        tokenResponse.setAccessToken(jsonObject.get("access_token").getAsString());
        tokenResponse.setIdToken(jsonObject.get("id_token").getAsString());
        tokenResponse.setExpiresIn(jsonObject.get("expires_in").getAsInt());

        if (jsonObject.has("refresh_token")) {
            tokenResponse.setRefreshToken(jsonObject.get("refresh_token").getAsString());
        }

        return tokenResponse;
    }

    public static UserClaims parseToken(String token) throws Exception {
        SignedJWT signedJWT = SignedJWT.parse(token);
        JWTClaimsSet claimsSet = signedJWT.getJWTClaimsSet();

        UserClaims userClaims = new UserClaims();
        userClaims.setSubject(claimsSet.getSubject());
        userClaims.setName((String) claimsSet.getClaim("name"));
        userClaims.setEmail((String) claimsSet.getClaim("email"));
        userClaims.setOid((String) claimsSet.getClaim("oid")); // Object ID

        // Extract roles from the token
        Object rolesClaim = claimsSet.getClaim("roles");
        if (rolesClaim != null) {
            if (rolesClaim instanceof List) {
                userClaims.setRoles((List<String>) rolesClaim);
            } else {
                userClaims.setRoles(Collections.singletonList((String) rolesClaim));
            }
        } else {
            userClaims.setRoles(Collections.emptyList());
        }

        // Parse custom claims if present
        Object groups = claimsSet.getClaim("groups");
        if (groups != null) {
            userClaims.setGroups((List<String>) groups);
        }

        return userClaims;
    }

    private static void renderErrorPage(String errorTitle, String errorMessage) {

        var container = Jt.container().key("error").use();
        Jt.title("Authentication Error").use(container);
        Jt.header("Authentication Failed").use(container);
        Jt.text("Error: " + errorTitle).use(container);
        Jt.text("Details: " + errorMessage).use(container);

        // Retry button
        if (Jt.button("Try Again").use(container)) {
            Jt.switchPage("/");
        }
    }
    /**
     * Model class for token response
     */
    public static class TokenResponse {
        private String accessToken;
        private String idToken;
        private String refreshToken;
        private int expiresIn;

        // Getters and setters
        public String getAccessToken() {
            return accessToken;
        }

        public void setAccessToken(String accessToken) {
            this.accessToken = accessToken;
        }

        public String getIdToken() {
            return idToken;
        }

        public void setIdToken(String idToken) {
            this.idToken = idToken;
        }

        public String getRefreshToken() {
            return refreshToken;
        }

        public void setRefreshToken(String refreshToken) {
            this.refreshToken = refreshToken;
        }

        public int getExpiresIn() {
            return expiresIn;
        }

        public void setExpiresIn(int expiresIn) {
            this.expiresIn = expiresIn;
        }
    }

    public static class UserClaims {
        private String subject;
        private String name;
        private String email;
        private String oid;
        private List<String> roles;
        private List<String> groups;

        // Getters and setters
        public String getSubject() {
            return subject;
        }

        public void setSubject(String subject) {
            this.subject = subject;
        }

        public String getName() {
            return name;
        }

        public void setName(String name) {
            this.name = name;
        }

        public String getEmail() {
            return email;
        }

        public void setEmail(String email) {
            this.email = email;
        }

        public String getOid() {
            return oid;
        }

        public void setOid(String oid) {
            this.oid = oid;
        }

        public List<String> getRoles() {
            return roles;
        }

        public void setRoles(List<String> roles) {
            this.roles = roles;
        }

        public List<String> getGroups() {
            return groups;
        }

        public void setGroups(List<String> groups) {
            this.groups = groups;
        }

        public boolean hasRole(String role) {
            return roles != null && roles.contains(role);
        }
    }

}
```

</Collapse>

## Create a web application in Microsoft Entra ID

Within Microsoft Entra ID in Azure, you'll need to register a new application and generate a secret needed to configure your app. In this example, your application will only accept personal Microsoft accounts, but you can optionally accept work and school accounts or restrict the application to your personal tenant. Microsoft Entra also lets you connect other, external identity providers.

### Register a new application

#### Step 1: Create a New Application Registration

1. Go to [Microsoft Azure](https://portal.azure.com/#home), and sign in to Microsoft.
1. Navigate to Microsoft Entra ID → App registrations → New registration
1. Fill in the application details:
   - Name: Your Dashboard Name (e.g., "Javelit Dashboard")
   - Supported account types: Choose based on your needs:
     - "Accounts in this organizational directory only" (single tenant)
     - "Accounts in any organizational directory" (multi-tenant)
   - Redirect URI: Select "Web" and enter: https://localhost:8501/auth/callback
1. Click Register

#### Step 2: Generate Client Secret

2. In your app registration, go to Certificates & secrets → Client secrets
2. Click New client secret
2. Add a description (e.g., "Dashboard Auth")
2. Set expiration (typically 24 months)
2. Click Add and copy the secret value immediately (you won't see it again)

#### Step 3: Configure API Permissions

3. Go to API permissions → Add a permission
3. Select Microsoft Graph
3. Choose Delegated permissions (for user authentication)
3. Add these permissions:
   - profile - Read user profile
   - email - Read email
   - openid - Sign in users
   - Directory.Read.All - To read user/group info
3. Click Grant admin consent

#### Step 4: Configure Application Roles (for Role-Based Access)

4. Go to App roles → Create app role
4. Create roles for your dashboard:
   - Display name: "Admin"
   - Value: admin
   - Description: "Dashboard Administrator"
   - Do you want to enable this app role?: Yes
4. Repeat for other roles (e.g., "Viewer", "Editor", etc.)

#### Step 5: Assign Roles to Users

5. Go to Enterprise applications → Find your app
5. Select Users and groups → Add user/group
5. Select users and assign them the appropriate roles

#### Step 6: Collect Your Credentials

6. Save these values (you'll need them in your Java code):

   - Client ID: Application (client) ID
   - Client Secret: The secret you created
   - Tenant ID: Directory (tenant) ID
   - Redirect URI: https://localhost:8501/auth/callback




## Build the example


### Configure your secrets

1. In `your_repository`, create a `.env` file.

1. Add `.env` to your `.gitignore` file.

   <Important>
      Never commit secrets to your repository. For more information about `.gitignore`, see [Ignoring files](https://docs.github.com/en/get-started/getting-started-with-git/ignoring-files).
   </Important>



1. In `.env`, add your connection configuration:

   ```bash
   ENTRA_CLIENT_ID=xxx
   ENTRA_CLIENT_SECRET=xxx
   ENTRA_TENANT_ID=xxx
   ENTRA_REDIRECT_URL=http://localhost:8501/auth/callback
   ```

   Replace the values of `ENTRA_CLIENT_ID`, `ENTRA_CLIENT_SECRET`, and `ENTRA_TENANT_ID` with the values you copied into your text editor earlier. 

1. Save your `.env` file.

### Initialize your app

1. In `your_repository`, create a file named `AppWithEntra.java`.
1. In a terminal, change directories to `your_repository`, and start your app:

   ```bash
   source .env
   javelit run -p 8501 AppWithEntra.java
   ```

   Your app will be blank because you still need to add code.

1. Copy the content of `AppWithEntra.java` from the [example](https://github.com/javelit/javelit/blob/main/examples/AppWithEntra.java) in the [examples](https://github.com/javelit/javelit/tree/main/examples) directory.


### Open your deployed app, and test it.
