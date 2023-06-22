# Step 1 - Create certificate profile

The first step is to create a certificate profile for server TLS certificates. The certificate profile defines the content and constraints of new certificates. For example, you can define which key types are allowed and what extensions to use in certificates. For an introduction to certificate profiles, see the Certificate Profiles Overview.

To create a certificate profile for server TLS certificates, do the following:




1. In EJBCA, under **CA Functions**, click **Certificate Profiles**.
The Manage Certificate Profiles page displays a list of available profiles.
2. Click **Clone** next to the **SERVER** template to use that as a basis for creating your new profile.
3. Name the new certificate profile **TLS Server Profile** and click **Create from template**.
4. To edit the profile values to fit your needs, find the newly created **TLS Server Profile** in the list and click **Edit**.
5. On the Edit page, verify that the type is End Entity and update the following:
- For **Available Key Algorithms**, select ECDSA to only allow elliptic curve keys.
- For **Available ECDSA curves**, select **P-256 / prime256v1 / secp256r1**.
- For **Signature Algorithm**, verify that Inherit from Issuing CA is selected.
- For **Validity or end date of the certificate**, specify **1y**.
- Expiration Restrictions: Enable to only allow certificates to expire on Tuesdays, Wednesdays, and Thursdays to avoid certificates expiring close to a weekend.
6. Under Permissions, you can allow the requester of the certificate to override certain default values of the profile that you are configuring, thus overriding profile defaults such as validity or extensions and setting their own values. By default, EJBCA does not allow any overrides and thus the certificate issued will be defined by what is configured in this profile.
7. The **X.509v3 extensions** section allows you to define the extensions added to the certificate:
- Clear Basic Constraints since this constraint defines that this is an end entity certificate and not a CA certificate, and you want this to be optional.
- For **Key Usage**, verify that **Digital Signature** and **Key encipherment** is selected.
- For **Extended Key Usage**, verify that Server Authentication is selected.
Extended key usages define how the certificate and the key pair can be used. If you need a key usage that is not available by default, you can add additional extended key usages in the EJBCA System Configuration, see Extended Key Usages.

- For **X.509v3 extensions - Names**, verify that **Subject Alternative Name** is selected and clear the **Issuer Alternative Name** extension as the CA does not have an alternative name.
- Under **X.509v3 extensions - Validation data**:
  - Enable **CRL Distribution Points** to allow validation later on.
  - Enable **Use CA defined CRL Distribution Point** to use the value pre-configured in the CA.
  - Enable **Authority Information Access** to use the locations configured in your CA settings:
    - Enable **Use CA defined OCSP locator** for where the OCSP services are available
    - Enable **Use CA defined CA issuer** for where the issuing CA certificate can be retrieved from.
8. Under **Other Data**, update the following:

- Clear **LDAP DN order** to disable the LDAP ordering of the DN attributes and use the standard X.509 ordering instead.
- For **Available CAs**, select your Sub CA MyPKISubCA-G1.

9. Click **Save** to store the certificate profile.

The newly created **TLS Server Profile** is displayed in the list of certificate profiles.
