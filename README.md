# Walkthrough: Add a sign-in-only custom policy in Azure Active Directory B2C

Active Directory Federation Services (ADFS) is an on-premise solution for enterprise identity federation, allowing you to permit the use of identities from multiple identity providers for multiple relying parties. ADFS also enables you to transform and augment claims, enforce service authorization, and integrates directly with Active Directory to provide an out-of-the-box identity provider with multi-factor authentication options.

Azure Active Directory B2B and B2C both build on the ADFS experience by easily federating with other Azure Active Directory instances, as well as popular social identity providers such as Google and Facebook. More significantly, Azure AD B2B and B2C provide many identity governance mechanisms that help you stay compliant with legal and regulatory requirements.

Azure AD B2B and B2C each require a shadow account for each user in their respective directories in order to fulfill the identity governance functions. Conversely, ADFS does not provide the identity governance functions, and does not need such shadow accounts. In all cases, a pre-configured technical trust needs to be established. But in terms of the default user experience, a B2B or B2C user needs to "sign-up" in order to successfully get access to services. ADFS on the other hand, does not force a "sign-up".

In this Azure AD B2C scenario, identity information is collected from the identity provider and passed on to the relying party without the need for the user to sign-up. As a result, Azure AD B2C does not persist any user information locally in its own directory, similar to ADFS.  While this eliminates the need to manage any local identities, it also prevents you from leveraging any identity governance capabilities of Azure AD B2C, meaning that identity governance must be implemented at the identity providers and/or the relying parties and/or in a custom REST API, all of which should be negotiated in advance of your implementation. Because data flows through your Azure AD B2C instance, you may be held accountable for any breaches or identity issues.

A sign-in-only custom policy has a very limited scope of purpose, where 
- Azure AD B2C itself does not provide any function not already provided by the identity providers and/or relying parties, and 
- you want to simplify home realm discovery for your relying parties and/or you need to inject a process in the authentication flow (via [REST API](https://docs.microsoft.com/en-us/azure/active-directory-b2c/custom-policy-rest-api-intro) e.g. such as external identity validation not done by the identity provider, or an authorization lookup to augment claims).

This sample federates Azure AD B2C with the Facebook identity provider. When a user signs in, Facebook claims are transformed into B2C claims, which are then sent to the relying party, all without the creation of a user identity in the B2C directory. While Facebook is used in the example, this policy can be adapted to work with any claims provider.

## Prerequisites

- Understand how the custom policy inheritance model works in [TrustFrameworkPolicy](https://docs.microsoft.com/en-us/azure/active-directory-b2c/trustframeworkpolicy)
- Complete the steps in [Get started with custom policies](https://docs.microsoft.com/en-us/azure/active-directory-b2c/custom-policy-get-started). 

## Review the existing configuration

You should have a working custom policy for sign-up and sign-in with non-local accounts (an external identity provider such as Facebook). The following steps illustrate how Azure AD B2C creates a local user account for each external identity on sign-up.

1. Sign in to the [Azure portal](https://portal.azure.com).
1. Make sure you're using the directory that contains your Azure AD tenant by selecting the **Directory + subscription** filter in the top menu and choosing the directory that contains your Azure AD tenant.
1. Choose **All services** in the top-left corner of the Azure portal, and then search for and select **App registrations**.
1. Select **Identity Experience Framework**.
1. Select the existing sign-up (or sign-up/sign-in) policy and click the **Run now** button.
1. Verify that you can sign-in using an external IdP. If you have not previously signed-up, B2C will tell you that you need to sign-up before you can sign-in.
1. When sign-in is complete, verify that the user you signed in with now exists as an entity in the B2C directory by selecting **Users** under the **Manage** menu of **Identity Experience Framework**.
1. Delete this account from the B2C directory for future testing.

## Add a claims transformation

As part of a sign-up flow, Azure AD B2C will automatically generate a random unique immutable object identifier (ObjectId) when it creates a new entity in its user directory. This ObjectId is created once and persisted, and reused whenever the user signs in for any downstream processes or claims that depend on a unique user key. For example, for OIDC relying parties, this ObjectId is typically used as the subject ("sub") claim. Because the sign-in flow through custom policy does not include a sign-up - and therefore doesn't auto-generate this identifier - an ObjectId property needs to be explicitly created to ensure that downstream processes are not negatively affected.

Creating a substitute ObjectId with a new GUID will not work. Since the value is not persisted in the B2C directory, a new GUID will be created on each sign-in, violating the immutable requirement. Instead, a unique and immutable identifier needs to be calculated from incoming claims. Further, that calculated value must be unique, and each constituent claim needs to be immutable to ensure the output value is also immutable.

In this example, the **CreateAlternativeSecurityId** claims transformation is used to calculate a new ObjectId, which includes the user identifier as provided by the IdP, and an IdP identifier. In combination, this should provide a unique value. Immutability is dependent on guarantees made by the IdP that the user identifier it provides is also immutable. This should be verified with each IdP you work with. Make this change to your *TrustFrameworkExtensions.xml* file.

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

## Create a claims transformation claims provider

If you already have a generalized claims provider for claims transformation, include a new **B2CUtil-CreateObjectId** technical profile in that claims provider as shown in the following example. Otherwise, create a new claims provider. This will provide the ability to trigger the **CreateObjectId** claims transformation. Make this change to your *TrustFrameworkExtensions.xml* file.

```xml
<ClaimsProvider>
  <DisplayName>B2C Utilities</DisplayName>
  <TechnicalProfiles>
    <TechnicalProfile Id="B2CUtil-CreateObjectId">
      <DisplayName>B2C Utility to Create ObjectId Claim</DisplayName>
      <Protocol Name="Proprietary" Handler="Web.TPEngine.Providers.ClaimsTransformationProtocolProvider, Web.TPEngine, Version=1.0.0.0, Culture=neutral, PublicKeyToken=null" />
      <OutputClaims>
        <OutputClaim ClaimTypeReferenceId="objectId" DefaultValue="unknown" />
      </OutputClaims>
      <OutputClaimsTransformations>
        <OutputClaimsTransformation ReferenceId="CreateObjectId" />
      </OutputClaimsTransformations>
    </TechnicalProfile>
  </TechnicalProfiles>
</ClaimsProvider>
```

## Add a new user journey

The default user journey reads and writes to AAD, pulling data and creating user accounts in the directory. Create a new **SignInOnly** user journey that uses the **B2CUtil-CreateObjectId** technical profile after the claims exchange with your identity providers. This example uses Facebook as the identity provider, but any number of claim providers can be configured for step 1 and step 2. Make this change to your *TrustFrameworkExtensions.xml* file.

```xml
<UserJourney Id="SignInOnly">

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

    <OrchestrationStep Order="3" Type="ClaimsExchange">
      <ClaimsExchanges>
        <ClaimsExchange Id="SignInOnlyExchange" TechnicalProfileReferenceId="B2CUtil-CreateObjectId" />
      </ClaimsExchanges>
    </OrchestrationStep>
    
    <OrchestrationStep Order="4" Type="SendClaims" CpimIssuerTechnicalProfileReferenceId="JwtIssuer" />

  </OrchestrationSteps>

  <ClientDefinition ReferenceId="DefaultWeb" />

</UserJourney>
```

If you are updating an existing user journey, you will need to:
1. Update your user journey to eliminate any reads and writes to AAD. Assess why those reads or writes are being done and determine how to best address them. It is possible that you will be unable to use a sign-in-only flow because of some downstream requirement for AAD information. This constraint was identified at the beginning of this article.
1. Inject the orchestration step where **ClaimsExchange/@Id** = **SignInOnlyExchange** into your user journey just before you send claims back to the client.

## Modify your relying party policy

Create a new relying party XML file and name it *SignInOnly.xml*. You can use the *SignUpOrSignIn.xml* file from the *SocialAccounts* folder of the [Custom Policy Starter Pack](https://github.com/Azure-Samples/active-directory-b2c-custom-policy-starterpack) as a template (be sure to update the **TenantId**, **PolicyId**, **PublicPolicyUri**, and the **TenantId** and **PolicyId** of the **BasePolicy**). Change the **ReferenceId** of **DefaultUserJourney** to **SignInOnly**, which is the name of the new user journey created in the previous step.

```xml
<DefaultUserJourney ReferenceId="SignInOnly" />
```

The following is an example of an OIDC relying party policy using the **SignInOnly** user journey.

```xml
<RelyingParty>
  <DefaultUserJourney ReferenceId="SignInOnly" />
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
1. Select **Upload Custom Policy**, and then upload the policy files that you changed: *TrustFrameworkExtensions.xml* and *SignInOnly.xml*. 
1. Select the sign-in policy that you uploaded, and click the **Run now** button.
1. Verify that you can sign-in using an external IdP.
1. When sign-in is complete, verify that the user you signed in with **does not exist** in the B2C directory by selecting **Users** under the **Manage** menu of **Identity Experience Framework**.
