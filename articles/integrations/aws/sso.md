# Enable SSO to the AWS Console

By integrating Auth0 with AWS, you'll allow your users to log in to AWS using any supported [identity provider](/identityproviders). 

## Configure Auth0

Log in to the [Management Dashboard](${manage_url}/#/applications), and create a new [Client](/client) (you can also use an existing Client if you'd like). On the **Addons** tab, enable the **SAML2 Web App** addon.

![](/media/articles/integrations/aws/addons.png)

You'll be asked to configure this add-on using the pop-up that appears immediately after you've enabled the SAML2 Web App. 

On the **Settings** tab, populate **Application Callback URL** with `https://signin.aws.amazon.com/saml` and paste the following SAML configuration code into **Settings**:

```js
{
  "audience": "https://signin.aws.amazon.com/saml",
  "mappings": {
    "email": "http://schemas.xmlsoap.org/ws/2005/05/identity/claims/emailaddress",
    "name": "http://schemas.xmlsoap.org/ws/2005/05/identity/claims/name",
  },
  "createUpnClaim": false,
  "passthroughClaimsWithNoMapping": false,
  "mapUnknownClaimsAsIs": false,
  "mapIdentities": false,
  "nameIdentifierFormat": "urn:oasis:names:tc:SAML:2.0:nameid-format:persistent",
  "nameIdentifierProbes": [
    "http://schemas.xmlsoap.org/ws/2005/05/identity/claims/emailaddress",
  ]
}
```

![](/media/articles/integrations/aws/configure.png)

Scroll to the bottom and click **Save**.

Click over to the **Usage** tab. You'll need to configure Auth0 as the identity provider (IdP) for AWS, which requires you to provide the appropriate metadata to AWS. You can obtain a file containing this information by clicking **Identity Provider Metadata**.

![](/media/articles/integrations/aws/idp-download.png)

## Configure AWS

At this point, you're ready to continue the configuration process from the AWS side.

Log in to AWS, and navigate to the [IAM console](https://console.aws.amazon.com/iam). Using the left-hand navigation menu, select **Identity Providers**. Click **Create Provider**. 

![](/media/articles/integrations/aws/create-provider.png)

Set the following parameters:

| Parameter | Description and Sample Value |
| - | - |
| Provider Type | The type of provider. Set as `SAML` |
| Provider Name | A descriptive name for the provider, such as `auth0SamlProvider` |
| Metadata Document | Upload the file containing the Auth0 metadata you downloaded in the previous step here. |

![](/media/articles/integrations/aws/aws-configure-provider.png)

Click **Next Step**. Verify your settings and click **Create** if everything is correct.

![](/media/articles/integrations/aws/create-provider-confirm.png)

To use the provider, you must create an IAM role using the provider in the role's trust policy. 

In the IAM console, navigate to [Roles](https://console.aws.amazon.com/iam/home#/roles). Click **Create New Role**.

![](/media/articles/integrations/aws/iam-new-role.png)

On the **Select role type** page, select **Role for identity provider access**. 

![](/media/articles/integrations/aws/select-role-type.png)

Click **Select** for the **Grant Web Single Sign-On (WebSSO) access to SAML providers** option. When prompted, set the provider you created above as the **SAML provider** and click **Next Step** to proceed.

![](/media/articles/integrations/aws/select-saml-provider-to-trust.png)

On the **Verify Role Trust** page, accept the **Policy Document** proposed (this policy tells IAM to trust the Auth0 SAML IdP). Click **Next Step**.

On **Attach Policy**, select the appropriate policies to attach to the role. These define the permissions that users granted this role will have with AWS. For example, to grant your users read-only access to IAM, filter for and select the `IAMReadOnlyAccess` policy. Click **Next Step**.

Finally, set the role name and review your settings. Provide values for the following parameters:

| Parameter | Definition | 
| - | - |
| Role name | A descriptive name for your role |
| Role description | A description of what your role is used for |

Review the **Trusted entities** and **Policies** information, then click **Create Role**.

![](/media/articles/integrations/aws/iam-review-role.png)

At this point, you'll have created the necessary role to associate with your provider.

## Map the AWS Role to a User

::: note
For an example of how to define a server-side rule for assigning a role in an advanced use case, see the [Amazon API Gateway tutorial](/integrations/aws-api-gateway).
:::

The **AWS roles** specified will be associated with an **IAM policy** that enforces the type of access allowed to a resource, including the AWS Consoles. To map an AWS role to a user, you'll need to create a [rule](/rules) for this purpose.

::: note
For more information on roles and policies, see [Creating IAM Roles](http://docs.aws.amazon.com/IAM/latest/UserGuide/roles-creatingrole.html).
:::

```js
function (user, context, callback) {

  user.awsRole = 'arn:aws:iam::951887872838:role/TestSAML,arn:aws:iam::951887872838:saml-provider/MyAuth0';
  user.awsRoleSession = 'eugeniop';

  context.samlConfiguration.mappings = {
    'https://aws.amazon.com/SAML/Attributes/Role': 'awsRole',
    'https://aws.amazon.com/SAML/Attributes/RoleSessionName': 'awsRoleSession'
  };

  callback(null, user, context);

}
```

In the code snippet above, `user.awsRole` identifies the AWS role and the IdP. The AWS role identifier comes before the comma, and the IdP identifier comes after the comma.

There are multiple ways by which you can obtain these two values. In the example above, both of these values are hard-coded into the rules. You might also store these values in the [user profile](/user-profile), or you might derive them using other attributes.

For example, if you're using Active Directory, you can map properties associated with users, such as `group` to the appropriate AWS role:

```js
var awsRoles = {
  'DomainUser': 'arn:aws:iam::951887872838:role/TestSAML,arn:aws:iam::95123456838:saml-provider/MyAuth0',
  'DomainAdmins': 'arn:aws:iam::957483571234:role/SysAdmins,arn:aws:iam::95123456838:saml-provider/MyAuth0'
};

context.samlConfiguration.mappings = {
  'https://aws.amazon.com/SAML/Attributes/Role': awsRoles[user.group],
  'https://aws.amazon.com/SAML/Attributes/RoleSessionName': user.name,
};
```

## Test Your Setup

You are now set up for single sign-on to AWS. You can find the `Identity Provider Login URL` on the [Management Dashboard](${manage_url}). Open up your [client](${manage_url}/#/clients) to the **SAML2 Addon** settings area, and click over to the **Usage** tab.

![](/media/articles/integrations/aws/idp-download.png)

To test the single sign-on, navigate to the URL indicated. You should be redirected to the Auth0 sign in page. If you successfully sign in, you'll be redirected again, this time to AWS.