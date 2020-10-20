# panos-and-certificates
various pieces of relevant info regarding the use of certificates for various use cases on PAN-OS



##############Setting up the Admin Interface for HTTPS###############

So, you've just set up a PAN-OS device, and *gasp, oh no* when you log in to the web interface you find that your connection is not secure, and the cert is invalid. Yeah that's not great, we would all prefer to avoid broadcasting things in plain text. If you go the Device (or on a Panorama, the Panorama tab) and then to Setup > Management > Secure Communication Settings, you will notice there is a cert type of pre-defined. yeah for many modern OS's and browsers, this is not going to cut it, so you need to cut a cert that is going to meet their requirements and get you onto HTTPS.

Note: I am doing this from a MacOS device 10.14.x.  I am writing instructions that should meet the demands of new MacOS and iOS cert requirements (and potentially chrome as well). If you are working on Windows, you're going to have to figure this out by yourself to some degree (or linux) but the principles of what you're doing will remain the same.

I have found the certificate management function with PAN-OS to be rudimentary and have struggled to find a way to edit certificates in such a way that it passes the needs of MacOS for example. As a result, and because I don't want to pay $$$ for CA certs as I go through this process as an individual, I will be doing this off of locally generated certs on my Mac. If you have a better suggestion, or know of an affordable cloud/SaaS PKI system you recommend, don't hesitate to let me know. I mainly do POCs and not longer term engagements, so building things out in more expensive, but better ways is slightly beyond my individual reach.

####down to business #####

First off, you're going to want a CA. On your Mac, open "KeyChain Access", and then in the top left of your screen go to Keychain access > Keychain Assistant > Create a Certificate Authority. For my example I put the following data points:

Name: PANOS_Example_CA
Identity Type: Self Signed Root CA
User Certificate: S/MIME

[x]  let me override defaults
[ ]  Make this CA the default

skip forward 2 steps, uncheck 'sign your invitation', and advance steps until you get to 'Key Usage Extensions for this CA'
-include the key usage extension, and make sure it has certificate signing included


Advance again until 'Extended Key Usage Extension for Users of This CA', and make sure everything is allowed (may not be best practice but I'm trying to get this working first and foremost)

And then advance and advance until you are done.

Go back to Keychain Access, select your new CA, right click > Get info > open the trust drop down and set "when using this certificate" to "always trust"

Now, Open the Keychain Assistant Again, and this time create a cert. In this case, we are now creating a Leaf cert, give the name as the IP or DNS hostname of your mgmt interface, and make sure to hit 'let me override defaults

Advance to Certificate Information

Name (Common Name): _your_fw_mgt_ip_or_dns_name_here
choose the Root CA you created above to sign with

advance to 'Extended Key Usage Extentsion'
include
extension is critical

Advance to 'Subject Alternate Name Extension' -> this part is important for new Mac/iOS operating systems, some browsers, etc. you may need to look up the specific requiremetns for your OS and browser if you get validation issues

-make sure you have the DNS name or IP addresss as the extension values

and create cert.



#########pause##########

You should now have a CA cert and a leaf from that CA. The CA cert should be 'trusted for this account' because you manually made it that way. The leaf cert should just be 'valid' since you created it from a trusted CA. If this is not the case, you either messed up, or thigns have changed since I wrote this so go back and review all of the above for any errors/ things you missed. If you followed my instructions perfectly, then sorry you probably need to google some info on making certs for your OS.

Also, if you have made it to this stage and things look good, you shouldn't have to repeat the above in the future. Just make a new leaf cert from your already existing CA cert in accordance with my guidance, and then start the process below


########bringing this to PAN-OS########

Ok, so now within keychain access, you are going to export both the example CA and the leaf cert so that they are in a directory you can access. Open the firewall management UI and go to Device > Certificate Managemetn > Certificates (or Panorama > Certificate Management > Certificates if you're doing this for Panorama)

Import first the root CA, and then the leaf. I would advise keeping the cert name in PAN-OS the same as on your device, just to make things clear for you. Make sure to have cert type is local, and file format is PKCS12. You should have had to use a passphrase to export the cert, so use it here to successfully import.

Now go down to SSL/TLS Service Profile and create one with the _leaf_ cert for your mgmt interface. 
Go to Certificate Profiles and create one for your Root CA.
Go to Setup > Management > General Settings, and also set the SSL/TLS Service Profile to the one you just created

#####commit#####


If everything has gone correctly, you should get a message that the web ui is going to restart. when it is done, boom you should be on HTTPS. Now, in order for other people to be on HTTPS, they also must have the root CA you create on their machine and trust it. If you want to deploy a solution like this for your whole org, you are going to need to actually get into the PKI game. This is beyond the scope of this guide.

Thanks for reading and if you find any erros / have suggestions for improvement let me know.
kpawlak@paloaltonetworks.com


########GlobalProtect POC cert steps#######

So go back to Keychain assistant, create a new leaf cert from your local CA, type VPN server, ovverride defaults

In the Certificate Information page, common name should be the ip/dns of the untrust interface / interface on which you will host the GP portal and/or gateway (make more than one cert if these are different interfaces or on different devices)
(make sure it is signed by a root CA you trust (probably the same you created earlier)

Extended Key Usage Extension - you can do any, but I always make sure to specifically flag 'ssl server auth' for potal web ui + any initial ssl interactions
Subject alternative name extension page - again we will put in the IP and/or DNS of the untrust interface
Now upload it to the firewall (after you export it from keychain access app

From there you will be using it in the normal place of certificates in the GP setup process, such as discussed here: https://docs.paloaltonetworks.com/globalprotect/8-1/globalprotect-admin/globalprotect-quick-configs/remote-access-vpn-authentication-profile.html#idedc68ee0-d39f-4d91-bcae-5409f57c4071

So import it to the firewall, create the appropriate SSL/TLS service profile, use it in GlobalProtect portal/gateway config

####commit####




##############################GlobalProtect Client Cert Validation - POC ################################

As in all the above sections, this guidance is suitable for small POC deployments. If you are going to do this configuration for an enterprise you should get assistance from a PKI solution/ deployment so that you can create and manage certificates in a secure, compliant, scalable way.

At this stage, what we need is a certificate to do VPN client work that will be accepted by the globalprotect portal/gateway

once again open our trusty keychain assistant, create a cert of identity type leaf, cert type VPN Client, override defaults

For PAN-OS purposes, the common name is used for identification. If you are trying to do per-user certificates or per-machine certs, then you do need to be concerned with this field as it will be used for User-ID and/or machine identification. For POC purposes, it is fine to give it a generic name (I chose gp_test1)

Advance to 'Extended Key Usage Extension, include 'any' and 'ssl client auth'
In this case we *should* not need a SAN as the firewall is inspecting this cert for auth, rather than the OS/browser. we do not have to concern ourselves with passign the checks of a specific client OS or browser

after cert is created, right click > new identify preference, and add the URL for the GP portal/gateway as the location. This is so that the GP client knows to supply this certificate for VPN client authentication when going to this IP/URL

This cert is signed by the root CA you started with, so on your device, you shoud have a Certificate Profile referencing this CA. Go to your GP Gateway config > Authentication, and change certificate profile to match that profile. Also change the client auth entry so that it needs both user credentials and the cert. The GP Gateway will now look for the presence of a cert for client VPN which is signed by the CA you started with.

****if you would like you can upload this client cert to the fw as well so other admins can download it. alternatively, create unique certs for each admin/testuser and then hand them out to those people *****

####commit#####

If you want to extend this type of deployment to a full enterprise, I also recommend you use an MDM solution of some kind in order to distribute the certs, such as this process with jamf: https://docs.jamf.com/technical-papers/jamf-pro/integrating-ad-cs/10.6.0/Distribute_Certificates_Using_Configuration_Profiles.html

Alternatively, you may embed the cert directly in the GP agent and distribute the agent packeage (with embedded cert) using your management tool of choice. this will accomplish the cert + user auth use case, but does not create unique certs per user or machine.



