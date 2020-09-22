# Walkthrough: Add a sign-in-only custom policy in Azure Active Directory B2C

Active Directory Federation Services (ADFS) is an on-premise solution for enterprise identity federation, allowing you to permit the use of identities from multiple identity providers for multiple relying parties. ADFS also enables you to transform and augment claims, enforce service authorization, and integrates directly with Active Directory to provide an out-of-the-box identity provider with multi-factor authentication options.

Azure Active Directory B2B and B2C both build on the ADFS experience by easily federating with other Azure Active Directory instances, as well as popular social identity providers such as Google and Facebook. More significantly, Azure AD B2B and B2C provide many identity governance mechanisms that help you stay compliant with legal and regulatory requirements.

Azure AD B2B and B2C each require a shadow account for each user in their respective directories in order to fulfill the identity governance functions. Conversely, ADFS does not provide the identity governance functions, and does not need such shadow accounts. In all cases, a pre-configured technical trust needs to be established. But in terms of the default user experience, a B2B or B2C user needs to "sign-up" in order to successfully get access to services. ADFS on the other hand, does not force a "sign-up".

In this Azure AD B2C scenario, identity information is collected from the identity provider and passed on to the relying party without the need for the user to sign-up. As a result, Azure AD B2C does not persist any user information locally in its own directory, similar to ADFS.  While this eliminates the need to manage any local identities, it also prevents you from leveraging any identity governance capabilities of Azure AD B2C, meaning that identity governance must be implemented at the identity providers and/or the relying parties, all of which should be negotiated in advance of your implementation. Because data flows through your Azure AD B2C instance, you may be held accountable for any breaches or identity issues.

This sample federates Azure AD B2C with the Facebook identity provider. When a user signs in, Facebook claims are transformed into B2C claims, which are then sent to the relying party, all without the creation of a user identity in the B2C directory. While Facebook is used in the example, this policy can be adapted to work with any claims provider.

## Prerequisites

- Complete the steps in [Get started with custom policies](custom-policy-get-started.md). You should have a working custom policy for sign-up and sign-in with local accounts.
- Learn how to [Integrate REST API claims exchanges in your Azure AD B2C custom policy](custom-policy-rest-api-intro.md).

## Prepare your environment

Ensure that you have a working set of policy files, including a claims provider and relying party.

1. Sign in to the [Azure portal](https://portal.azure.com).
1. Make sure you're using the directory that contains your Azure AD tenant by selecting the **Directory + subscription** filter in the top menu and choosing the directory that contains your Azure AD tenant.
1. Choose **All services** in the top-left corner of the Azure portal, and then search for and select **App registrations**.
1. Select **Identity Experience Framework**.
1. Select the existing sign-in policy and click the **Run now** button.
1. Verify that you can sign-in using an external IdP.
1. When sign-in is complete, verify that the user you signed in with now exists as an entity in the B2C directory by selecting **Users** under the **Manage** menu of **Identity Experience Framework**.
1. Backup your original policy files *TrustFrameworkExtensions.xml* and *SignUpOrSignIn.xml*.

## Add a claims transformation

As part of a sign-up flow, Azure AD B2C will automatically generate a random unique immutable object identifier (ObjectId) when it creates a new entity in its user directory. This ObjectId is created once and persisted, and reused whenever the user signs in for any downstream processes or claims that depend on a unique user key. For example, for OIDC relying parties, this ObjectId is typically used as the subject ("sub") claim. Because the sign-in flow through custom policy does not include a sign-up - and therefore doesn't auto-generate this identifier - an ObjectId property needs to be explicitly created. 

Creating a substitute ObjectId with a new GUID will not work. Since the value is not persisted in the B2C directory, a new GUID will be created on each sign-in, violating the immutable requirement. Instead, a unique and immutable identifier needs to be calculated from incoming claims. Further, that calculated value must be unique, and each constituent claim needs to be immutable to ensure the output value is also immutable.

In this example, the CreateAlternativeSecurityId claims transformation is used to calculate a new ObjectId, and includes the user identifier as provided by the IdP, and an IdP identifier. In combination, this should provide a unique value. Immutability is dependent on guarantees made by the IdP that the user identifier it provides is also immutable.

```xml
<ClaimsTransformation Id="CreateObjectId" TransformationMethod="CreateAlternativeSecurityId">
  <InputClaims>
    <InputClaim ClaimTypeReferenceId="issuerUserId" TransformationClaimType="key" />
    <InputClaim ClaimTypeReferenceId="identityProvider" TransformationClaimType="identityProvider" />
  </InputClaims>
  <OutputClaims>
    <OutputClaim ClaimTypeReferenceId="objectId" TransformationClaimType="alternativeSecurityId" />
  </OutputClaims>
</ClaimsTransformation>
```

## Modify your claims provider

Update the claims provider to ensure that the new ObjectId is created. Include the following **OutputClaimsTransformations** in the relevant technical profile of your chosen claims provider.

```xml
<OutputClaimsTransformations>
  <OutputClaimsTransformation ReferenceId="CreateObjectId" />
</OutputClaimsTransformations>
```

The following is an example using the **Facebook-OAUTH** technical profile.

```xml
<ClaimsProvider>
  <DisplayName>Facebook</DisplayName>
  <TechnicalProfiles>
    <TechnicalProfile Id="Facebook-OAUTH">
      <OutputClaimsTransformations>
        <OutputClaimsTransformation ReferenceId="CreateObjectId" />
      </OutputClaimsTransformations>
    </TechnicalProfile>
  </TechnicalProfiles>
</ClaimsProvider>
```

As an alternative, you can create a separate technical profile to preserve your original technical profile. The following is an example for Facebook. The **TechnicalProfile** **Id** is different than the original **Facebook-OAUTH** value.

```xml
<ClaimsProvider>
  <Domain>facebook.com</Domain>
  <DisplayName>Facebook</DisplayName>
  <TechnicalProfiles>
    <TechnicalProfile Id="Facebook-OAUTH-sift">
      <DisplayName>Facebook</DisplayName>
      <Protocol Name="OAuth2" />
      <Metadata>
        <Item Key="ProviderName">facebook</Item>
        <Item Key="authorization_endpoint">https://www.facebook.com/dialog/oauth</Item>
        <Item Key="AccessTokenEndpoint">https://graph.facebook.com/oauth/access_token</Item>
        <Item Key="HttpBinding">GET</Item>
        <Item Key="UsePolicyInRedirectUri">0</Item>
        <Item Key="AccessTokenResponseFormat">json</Item>
      </Metadata>
      <CryptographicKeys>
        <Key Id="client_secret" StorageReferenceId="B2C_1A_FacebookSecret" />
      </CryptographicKeys>
      <InputClaims />
      <OutputClaims>
        <OutputClaim ClaimTypeReferenceId="issuerUserId" PartnerClaimType="id" />
        <OutputClaim ClaimTypeReferenceId="givenName" PartnerClaimType="first_name" />
        <OutputClaim ClaimTypeReferenceId="surname" PartnerClaimType="last_name" />
        <OutputClaim ClaimTypeReferenceId="displayName" PartnerClaimType="name" />
        <OutputClaim ClaimTypeReferenceId="email" PartnerClaimType="email" />
        <OutputClaim ClaimTypeReferenceId="identityProvider" DefaultValue="facebook.com" AlwaysUseDefaultValue="true" />
        <OutputClaim ClaimTypeReferenceId="authenticationSource" DefaultValue="socialIdpAuthentication" AlwaysUseDefaultValue="true" />
      </OutputClaims>
      <OutputClaimsTransformations>
        <OutputClaimsTransformation ReferenceId="CreateObjectId" />
        <OutputClaimsTransformation ReferenceId="CreateAlternativeSecurityId" />
      </OutputClaimsTransformations>
      <UseTechnicalProfileForSessionManagement ReferenceId="SM-SocialLogin" />
    </TechnicalProfile>
  </TechnicalProfiles>
</ClaimsProvider>
```

Note that any technical profiles referenced by the claims provider might be reading or writing to AAD. To create the sign-in flow through experience, these operations must be avoided. Assess your need for these operations and ensure that they are addressed (removed or provided for in another way).

## Add a new user journey

The default user journey reads and writes to AAD, pulling data and creating user accounts in the directory. Create a new user journey that uses the modified technical profile (using the **Facebook-OAUTH** technical profile in the example below) and does not refer to any AAD operations.

```xml
<UserJourney Id="SignInFlowThrough">

  <OrchestrationSteps>

    <OrchestrationStep Order="1" Type="ClaimsProviderSelection" ContentDefinitionReferenceId="api.idpselections">
      <ClaimsProviderSelections>
        <ClaimsProviderSelection TargetClaimsExchangeId="FacebookExchange" />
      </ClaimsProviderSelections>
    </OrchestrationStep>

    <OrchestrationStep Order="2" Type="ClaimsExchange">
      <ClaimsExchanges>
        <ClaimsExchange Id="FacebookExchange" TechnicalProfileReferenceId="Facebook-OAUTH" />
      </ClaimsExchanges>
    </OrchestrationStep>

    <OrchestrationStep Order="3" Type="SendClaims" CpimIssuerTechnicalProfileReferenceId="JwtIssuer" />

  </OrchestrationSteps>

  <ClientDefinition ReferenceId="DefaultWeb" />

</UserJourney>
```

## Modify your relying party policy

Edit your relying party XML file, changing the **ReferenceId** of **DefaultUserJourney** to **SignInFlowThrough**, which is the name of the new user journey created in the previous step.

```xml
<DefaultUserJourney ReferenceId="SignInFlowThrough" />
```

The following is an example of a relying party policy using the **SignInFlowThrough** user journey.

```xml
<RelyingParty>
  <DefaultUserJourney ReferenceId="SignInFlowThrough" />
  <TechnicalProfile Id="PolicyProfile">
    <DisplayName>PolicyProfile</DisplayName>
    <Protocol Name="OpenIdConnect" />
    <OutputClaims>
      <OutputClaim ClaimTypeReferenceId="displayName" />
      <OutputClaim ClaimTypeReferenceId="givenName" />
      <OutputClaim ClaimTypeReferenceId="surname" />
      <OutputClaim ClaimTypeReferenceId="email" />
      <OutputClaim ClaimTypeReferenceId="objectId" PartnerClaimType="sub"/>
      <OutputClaim ClaimTypeReferenceId="identityProvider" />
      <OutputClaim ClaimTypeReferenceId="tenantId" AlwaysUseDefaultValue="true" DefaultValue="{Policy:TenantObjectId}" />
    </OutputClaims>
    <SubjectNamingInfo ClaimType="sub" />
  </TechnicalProfile>
</RelyingParty>
```

## Test the custom policy

1. Sign in to the [Azure portal](https://portal.azure.com).
1. Make sure you're using the directory that contains your Azure AD tenant by selecting the **Directory + subscription** filter in the top menu and choosing the directory that contains your Azure AD tenant.
1. Choose **All services** in the top-left corner of the Azure portal, and then search for and select **App registrations**.
1. Select **Identity Experience Framework**.
1. Select **Upload Custom Policy**, and then upload the policy files that you changed: *TrustFrameworkBase.xml*, and *TrustFrameworkExtensions.xml*, *SignUpOrSignin.xml*, *ProfileEdit.xml*, and *PasswordReset.xml*. 
1. Select the sign-up or sign-in policy that you uploaded, and click the **Run now** button.
1. You should be able to sign up using an email address or a Facebook account.
1. The token sent back to your application includes the `balance` claim.

```json
{
  "typ": "JWT",
  "alg": "RS256",
  "kid": "X5eXk4xyojNFum1kl2Ytv8dlNP4-c57dO6QGTVBwaNk"
}.{
  "exp": 1584961516,
  "nbf": 1584957916,
  "ver": "1.0",
  "iss": "https://contoso.b2clogin.com/f06c2fe8-709f-4030-85dc-38a4bfd9e82d/v2.0/",
  "aud": "e1d2612f-c2bc-4599-8e7b-d874eaca1ee1",
  "acr": "b2c_1a_signup_signin",
  "nonce": "defaultNonce",
  "iat": 1584957916,
  "auth_time": 1584957916,
  "name": "Emily Smith",
  "email": "emily@outlook.com",
  "given_name": "Emily",
  "family_name": "Smith",
  "balance": "202.75"
  ...
}
```

## Next steps

To learn how to secure your APIs, see the following articles:

- [Walkthrough: Integrate REST API claims exchanges in your Azure AD B2C user journey as an orchestration step](custom-policy-rest-api-claims-exchange.md)
- [Secure your RESTful API](secure-rest-api.md)
- [Reference: RESTful technical profile](restful-technical-profile.md)
