https://doc.primekey.com/ejbca/tutorials-and-guides/tutorial-create-a-pki-hierarchy-in-ejbca

# Step 1 - Create certificate profiles

The first step towards creating a CA hierarchy is to create certificate profiles for the Root CA and Sub CA. The certificate profile defines the constraints of new certificates, for example, what keys it can use, and what the extensions will be.

## Create Root CA certificate profile
The following provides steps for creating a Root CA certificate profile by cloning the MyRootCAProfile certificate profile.

![Create_ROOT_CA_Profile](https://github.com/s0p4L1N/ejbca-docker-nginx/assets/92848369/13b39428-dc7f-464f-a927-983ee917e94b)
  
Description of the GIF below:

1. In EJBCA, under **CA Functions**, click **Certificate Profiles**.
The Manage Certificate Profiles page displays a list with default profiles and the Root CA Profile created in the previous tutorial Create your first Root CA using EJBCA.

2. Click **Clone** next to the **MyRootCAProfile** template to use that as a basis for creating your new Root CA profile.

3. Name the new certificate profile **MyPKIRootCAProfile**, and click **Create from template**.

4. To edit the profile values to fit your needs, find the newly created **MyPKIRootCAProfile** displayed in the list and click **Edit**.

5. On the Edit page, update the following to use elliptic curve keys instead of RSA keys:
- For **Available Key Algorithms**, select ECDSA.
- For **Available ECDSA curves**, select P-256 / prime256v1 / secp256r1.

7. Click **Save** to store the Root CA certificate profile.

The newly created MyPKIRootCAProfile is displayed in the list of certificate profiles.




## Create Sub CA certificate profile

To create a certificate profile for creating the Sub CA in a later step, do the following:

![Create_SUB_CA_Profile](https://github.com/s0p4L1N/ejbca-docker-nginx/assets/92848369/27665233-b05e-4365-932b-55f0953857c3)

1. On the EJBCA Manage Certificate Profiles page, click **Clone** by the SUBCA default template to create a new profile using that template.

2. Name the new certificate profile **MyPKISubCAProfile**, and click Create from template.

3. To edit the profile values to fit your needs, find the newly created **MyPKISubCAProfile** displayed in the list and click **Edit**.

4. On the Edit page, update the following to use elliptic curve keys instead of RSA keys:
   
- For **Available Key Algorithms**, select **ECDSA**.
- For **Available ECDSA curves**, select P-256 / prime256v1 / secp256r1.
- For **Signature Algorithm**, verify that Inherit from Issuing CA is selected.
- For **Validity or end date of the certificate**, specify **15y**.
- Under **X.509v3 extensions**, update the following:
- Select **Path Length Constraint** and set it to 0 to ensure that this Sub CA cannot issue any further sub CAs beneath it and is only allowed to issue end entity certificates.
- Under **X.509v3 extensions - Validation data**, update the following:
- Enable **CRL Distribution Points**.
- Enable **Use CA defined CRL Distribution Point** to allow setting a value for the distribution point in your Root CA.
- Enable **Authority Information Access**.
- Enable **Use CA defined OCSP locator**.
- Enable **Use CA defined CA issuer**.
- Under **Other Data**, clear **LDAP DN order**.
5. Click **Save** to store the Sub CA certificate profile.
  
The newly created **MyPKISubCAProfile** is displayed in the list of certificate profiles.

# Step 2 - Create Crypto Tokens

## Create Root CA crypto token
To create a soft Root CA crypto token and keys, follow these steps:  

![Create_ROOT_CA_Cryptotoken](https://github.com/s0p4L1N/ejbca-docker-nginx/assets/92848369/953bf8bf-8416-4c46-bd80-7dae42c5b114)


1. In the EJBCA menu, under **CA Functions**, click **Crypto Tokens**.

2. Click **Create new** and specify the following on the New Crypto Token page:
- **Name:** Name the Root CA crypto token **MyPKIRootCACryptoToken**.
- **Authentication Code:** Enter a password to be used to activate the crypto token if the container is restarted. Remember this password.

3. Click **Save** to create the Root CA crypto token.

4.Next, generate three CA keys:
- In the Name field that says **signKey**, specify **myPkiRootCaSignKey0001**, select **P-256 / prime256v1 / secp256r1** as defined in the certificate profile, and then click Generate new key pair to create the keys.
- Repeat to create the default encryption key: name the key **myPkiRootCaEncryptKey0001**, select **RSA 4096**, and then click **Generate new key pair**. 
- Last, repeat to create a test key: name the key **testKey**, select **P-256 / prime256v1 / secp256r1** as the CA signing key is using, and then click **Generate new key pair**.

You have now created the Root CA crypto token and keys.


## Create Sub CA crypto token

Repeat the same steps than ROOT CA CryptoToken but change the name of the crypto token to COMPANY SUB CA CryptoToken.

# Step 3 - Create CAs

## Create Root CA

To create the Root CA, follow these steps:

![Create_ROOT_CA_Cert](https://github.com/s0p4L1N/ejbca-docker-nginx/assets/92848369/aa943478-1f82-4439-bfa6-d420cc515178)


1. Click **Certification Authorities** under **CA Functions**.
2. In the **Add CA field**, enter the name “MyPKIRootCA-G1” and click **Create**.
3. On the Create CA page, update the following:
- Select the Root CA crypto token **MyPKIRootCACryptoToken** (created earlier in Step 2 - Create crypto token) in the **Crypto Token** list.
- For **Signing Algorithm**, select SHA512withECSDSA.
- Map the keys for their intended usages: the certSignKey and keyEncryptKey keys are automatically selected with the keys you created, for **defaultKey**, select your **myPkiRootCaEncryptKey0001**.
- Under **CA Certificate Data**, specify the following:
  - **Subject DN:** Enter "CN = My PKI Root CA - G1, O = Keyfactor Community, C = SE".
  - **Signed By:** Self Signed.
  - **Certificate Profile:** Select the MyPKIRootCAProfile created in Step 1 - Create certificate profiles.
  - **Validity:** Specify 30y.
  - **LDAP DN order**: Clear **Use**.
- Under CRL Specific Data, specify the following:
  - **CRL Expire Period**: Update to a CRL lifetime of 3 months by entering 3mo.
  - **CRL Overlap Time**: Set to 0m as automatic CRL issuing is not used.
-  Under Default CA defined validation data, define default values to be used in certificate profiles that are issued by the CA:
  - **Default CRL Distribution Point**: http://my.pki/crls/MyPKIRootCA-G1.crl
  - **OCSP service Default URI**: http://my.pki/ocsp
  - **CA issuer Default URI**: http://my.pki/certs/MyPKIRootCA-G1.crt
  
4. Click **Create** to create the Root CA.

The created MyPKIRootCA-G1 is displayed in the list of CAs.


## Create Sub CA
To create the Sub CA, follow these steps:

![Create_SUB_CA_Cert](https://github.com/s0p4L1N/ejbca-docker-nginx/assets/92848369/6c8716f3-a261-461e-acb9-d16bbab3be36)


1. Click **Certification Authorities** under **CA Functions**.
2. In the **Add CA** field, enter the name “MyPKISubCA-G1” and click **Create**.
3. On the Create CA page, update the following:
- Select the Sub CA crypto token **MyPKISubCACryptoToken** (created earlier in Step 2 - Create crypto token) in the Crypto Token list.
- For Signing Algorithm, select SHA256withECSDSA.
- Map the keys for their intended usages: the certSignKey and keyEncryptKey keys are automatically selected with the keys you created, for **defaultKey**, select your **myPkiSubCaEncryptKey0001**.
- Under **CA Certificate Data**, specify the following:
  - **Subject DN**: Enter "CN = My PKI Sub CA - G1, O = Keyfactor Community, C = SE".
  - **Signed By**: Select MyPKIRootCA-G1 to have it signed by the local Root CA.
  - **Certificate Profile**: Select the MyPKISubCAProfile created in Step 1 - Create certificate profiles.
  - **Validity**: Specify **15y**.
  - **LDAP DN order**: Clear **Use**.
- Under CRL Specific Data, specify the following:
  - **CRL Expire Period**: Update to a CRL lifetime of 3 days by entering **3d**.
  - **CRL Issue Interval**: Enter **1d** to enable automatic CRL issuing and for a new CRL to be created one day after the previous CRL was created.
  - **CRL Overlap Time**: Set to **0m** since the issue interval is already set and there is no need to specify the overlap which is the time in advance of the expiring a new CRL should be created.
- Under Default CA defined validation data, define default values to be used in certificate profiles that are issued by the CA:
  - **Default CRL Distribution Point**: http://my.pki/crls/MyPKISubCA-G1.crl
  - **OCSP service Default URI**: http://my.pki/ocsp
  - **CA issuer Default URI**: http://my.pki/certs/MyPKISubCA-G1.crt
4. Click **Create** to create the Root CA.

The created MyPKISubCA-G1 is displayed in the list of CAs.


# Next steps

In this tutorial, you learned how to create certificate profiles, crypto tokens and keys, and bring that information together to create the Root CA and Sub CA in a two-tier public key infrastructure (PKI) hierarchy.

Next, you can start issuing end entity certificates following the tutorial [Issue TLS server certificates with EJBCA.](https://github.com/s0p4L1N/ejbca-docker-nginx/blob/main/tutorial-issue-tls-server-certificates-with-ejbca.md)
