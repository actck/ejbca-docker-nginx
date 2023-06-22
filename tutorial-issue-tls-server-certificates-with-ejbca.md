# Step 1 - Create certificate profile

The first step is to create a certificate profile for server TLS certificates. The certificate profile defines the content and constraints of new certificates. For example, you can define which key types are allowed and what extensions to use in certificates. For an introduction to certificate profiles, see the Certificate Profiles Overview.

To create a certificate profile for server TLS certificates, do the following:

![Create_TLS_Server_Cert_Profile](https://github.com/s0p4L1N/ejbca-docker-nginx/assets/92848369/27b1c2b9-cdf1-4c53-8bac-e8f909f0e8a5)


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



# Step 2 - Create end entity profile

Next, create an end entity profile that allows you to define what information about holders of certificates EJBCA keeps track of and adds as subject information.

To create an end entity profile, follow these steps:

![Create_TLS_End_Entities_Profile](https://github.com/s0p4L1N/ejbca-docker-nginx/assets/92848369/7ed5d3ff-00e9-4c41-8dc8-bfa6994c5349)


1. In EJBCA, under **RA Functions**, click **End Entity Profiles**.
2. In the **Add Profile** field, add a name for the new profile, in this example **TLS Server Profile**, and click **Add profile**.
3. Select the newly created **TLS Server Profile**, and click **Edit End Entity Profile** to update the profile.
4. Edit the profile and update the following:
- Clear **End Entity E-mail** to not save any email addresses.
- Under **Subject DN Attributes**, specify the following:
  - For **Subject DN Attributes**, add an Organization field and a Country field.
  - For **CN, Common Name**, verify that Required and Modifiable are selected to allow the value to be used when a new request is made. You can optionally add a Validation with a regex to restrict what values are allowed.
  - For **O, Organization**, verify that Required is selected and clear Modifiable and enter "Keyfactor Community" to be used as the organization.
  - For **C, Country**, verify that Required is selected and clear Modifiable and enter "SE" to use Sweden as the country.
- Under **Other Subject Attributes**, you can specify options for Subject Alternative Name (SAN). Some browsers require this field and it should be implemented for server certificates:
  - In the **Subject Alternative Name** list, select **DNS Name** and click **Add**. For the displayed **DNS Name** field, select **Use entity CN field** to add a DNS name that is the same as your common name field.
  - Add additional optional DNS names to allow more than one DNS name in the certificates. Select **DNS Name** and click **Add**. For the displayed **DNS Name** field, leave the default **Modifiable** option selected. Repeat to add additional optional DNS names.
- Main **Certificate Data** allows you map the profile to be used together with default certificate profiles and CAs:
  - For **Default Certificate Profile**, select the TLS Server Profile you created in Step 1 - Create certificate profile).
  - For **Default CAs**, select the MyPKISubCA-G1 to restrict this profile to only be usable by your Sub CA (created in Create a PKI Hierarchy in EJBCA).
- Specify **Default Token** options to define how the key pair generation should be implemented for the certificates:

  a. Select **User Generated** and CA-side key pair generation in **PEM file** format to allow the CA to generate the key pair together with the certificate.

User Generated means that the requester generates their own key pair and thus that the user creates and provides a certificate signing request (CSR) for the certificate request to EJBCA. The other file options allow the CA to generate the private key and certificate and return those to the requester as a single file in the selected format.

5. Click **Save** to store the end entity profile


# Step 3 - Issue server certificate

To issue a server certificate, use the EJBCA RA web interface to make a new request and enroll using your new profiles.

![Create_TLS_Enroll_Certificate](https://github.com/s0p4L1N/ejbca-docker-nginx/assets/92848369/3d9ebabf-c9e3-4535-8c17-a9a729d7746c)


1. In EJBCA, click **RA Web** to access the EJBCA RA UI.
2. Select **Make New Request** from the** Enroll** menu.
3. For **Certificate Type**, select your TLS Server Profile (created in Step 1 - Create certificate profile).
4. For **Key-pair generation**, select Provided by CA to allow you to generate it directly from EJBCA
5. In the Provide request info, add the Common Name of the target server in the CN, Common Name field.
6. In optionnal Subject Alternative Name Attributes, add the DNS Name of the target server
7. In confirm request, verify all the informations are correct.
8. Click **Download** PEM for Linux oriented, or PKCS12 for Windows oriented server.

The downloaded file includes your certificate server and CA certificates for the Root CA and Sub CA.
You have now PEM file that you can import on your server and use it with NGINX or Apache Web Server.
