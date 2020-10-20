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


########bringing this to PAN-OS########

Ok, so now within keychain access, you are going to export both the example CA and the leaf cert so that they are in a directory you can access. Open the firewall management UI and go to Device > Certificate Managemetn > Certificates (or Panorama > Certificate Management > Certificates if you're doing this for Panorama)

Import first the root CA, and then the leaf. I would advise keeping the cert name in PAN-OS the same as on your device, just to make things clear for you. Make sure to have cert type is local, and file format is PKCS12. You should have had to use a passphrase to export the cert, so use it here to successfully import.

Now go down to SSL/TLS Service Profile and create one with the _leaf_ cert for your mgmt interface. 

Then go our to Setup > Management > Secure Communication Settings. Change cert type to local

#####commit#####







