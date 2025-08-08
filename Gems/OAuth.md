  
Look this video : 

[**https://www.youtube.com/watch?v=hcSG5ktkasg&ab_channel=SoftwareArchitectureandDesign**](https://www.youtube.com/watch?v=hcSG5ktkasg&ab_channel=SoftwareArchitectureandDesign)

**  
  
Grant Types: Based on the nature of app we use the different  

- **Notes —> Understand the auth code with pkce you will understand all….**

- **Client-credentials. -> Machine to Machine / Server to Server / B2B (no user involved)**
- Resource Owner **Password -> If the user trust the app it provides credentials to app itself which is running on server**
- **Authorization code —> If user does not trust the  app it provide the credentials to the Identity Server (Like Okta/Google) and get the Authorization code which it forwards to the app and app request to Identity Server with AC and get the access token.**
- **Implicit —> those app which is running in the front which does not have storage and cannot manage the client id and client secret then instead of authorization code user authenticates from the identity server and get the access code and send it directly to the application. This is highly risky** 
- **Authorization code with PKCE —> here the app generate code verifier and calculate the code challenge and send it along with the user credentials user present the code challenge that is sent by application to the identity server. The application send the authorization code along with code verifier to the identity server and get the access token.  
      
    ![[Does the Application.jpeg]]

  

  ![[Client Credentials.png]]

  

  ![[Password.png]]

  ![[Authorization Code.png]]

  

  
![[Authorization Code 1.png]]
  

  ![[Pasted image 20250430114350.png]]![[Authorization Code with PKCE.png]]![[Authorization Code with PKCE 1.png]]


![[Authentication.png]]
## **Refresh Tokens**

### 📌 Use Case: Long-lived sessions

- Used to obtain new access tokens **without user interaction**.
    
- Sent securely to the token endpoint in exchange for a new access token.
    

#### 📄 Interview Script:

**Q: What’s the role of refresh tokens in OAuth?**  
"Access tokens are short-lived for security. Refresh tokens allow the client to request a new access token when the original expires, without prompting the user again."


