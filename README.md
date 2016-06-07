# HOWTO for generating self-signed certificates

NOTE If you don't want to use the default OpenSSL version due to obsolesence 
or known security issues, install a newer version (LibreSSL recommended) and 
put it ahead of the default openssl command on your path.

Those who prefer to use a GUI tool for managing keys and keystores should consider
[KeyStore Explorer](http://www.keystore-explorer.org/).

Generate a Key Passphrase
-------------------------

Either make one up, or [roll the dice](https://www.random.org/dice/?num=5) 
then pick strings using [diceware](http://world.std.com/~reinhold/diceware.html).

The examples below assume you stored the key passphrase in a file named keypass.txt

Generate a private key
-------------------------

    cat keypass.txt | \
    $(which openssl) genrsa -aes256 -passout fd:0 -out host.key 2048


Generate an openssl config file
-------------------------------
Specify subject common name (CN) and any subject alternate names (subjAltName)

Here is an example


    # TLS server certificate request

    # This file is used by the openssl req command. The subjectAltName cannot be
    # prompted for and must be specified in the SAN environment variable.
    
    [ default ]
    SAN                             = DNS:yourdomain.tld    # Default value
    
    [ req ]
    default_bits                    = 2048                  # RSA key size
    encrypt_key                     = no                    # Protect private key
    default_md                      = sha256                # MD to use
    utf8                            = yes                   # Input is UTF-8
    string_mask                     = utf8only              # Emit UTF-8 strings
    prompt                          = yes                   # Prompt for DN
    distinguished_name              = server_dn             # DN template
    req_extensions                  = server_reqext         # Desired extensions
    
    [ server_dn ]
    0.domainComponent               = "1. Domain Component            (eg, com)       "
    1.domainComponent               = "2. Domain Component            (eg, company)   "
    2.domainComponent               = "3. Domain Component            (eg, pki)       "
    organizationName                = "4. Organization Name           (eg, company)   "
    organizationName_default        = "UBAR Corporation"
    organizationalUnitName          = "5. Organizational Unit Name    (eg, section)   "
    organizationalUnitName_default  = "Web Servers"
    commonName                      = "6. Common Name                 (eg, FQDN)      "
    commonName_max                  = 64
    countryName                     = "7. Country Name                (2 letter code) "
    countryName_default             = "US"
    stateOrProvinceName             = "8. State or Province Name      (full name)     "
    stateOrProvinceName             = "California"
    localityName                    = "9. Locality Name               (eg, city)      "
    
    [ server_reqext ]
    basicConstraints                = CA:false
    keyUsage                        = nonRepudiation,digitalSignature,keyEncipherment
    extendedKeyUsage                = serverAuth,clientAuth
    subjectKeyIdentifier            = hash
    subjectAltName                  = @alt_names
    
    [alt_names]
    DNS.1 = $ENV::DNS_1
    DNS.2 = $ENV::DNS_2
    DNS.3 = $ENV::DNS_3
*host-cert.cnf*


Generate a certificate request
------------------------------

    cat keypass.txt | \
    SAN=DNS:ubar.com,DNS:f.ubar.com \
    DNS_1='f.ubar.com' \
    DNS_2='eng.ubar.com' \
    DNS_3='www.ubar.com' \
    bash -c '$(which openssl) req -new \
      -config etc/host-cert.cnf \
      -subj "/DC=com/DC=ubar/DC=f/O=UBAR CO/OU=Engineering/CN=f.ubar.com/C=US/ST=CA/L=San\ Francisco" \
      -passout fd:0 \
      -out host.csr \
      -keyout host.key'


View certifcate request to confirm it's OK
------------------------------------------

    $(which openssl) req -text -noout -verify -in host.csr

Make sure the subject, usages and DNS entries look OK


Generate a self-signed certificate
----------------------------------

    $(which openssl) x509 -req \
      -sha256 \
      -days 3650 \
      -in host.csr \
      -signkey host.key \
      -out host.crt


View self-signed certifcate to confirm it's OK
------------------------------------------

    $(which openssl) x509 -in host.crt -text -noout

Make sure the Subject and Issuer match and look OK


Strip the passphrase from the private key
-----------------------------------------

    $(which openssl) rsa -in host.key -out host-nopass.key


Generate a password for PKCS12 file
-----------------------------------

Use the same process as generating the key phrase, then store the PKCS12 password in p12pass.txt


Convert x509 cert to PKCS12
---------------------------

    cat p12pass.txt | \
    $(which openssl) pkcs12 -export \
      -passout fd:0 \
      -in host.crt \
      -inkey host.key \
      -out host.p12 \
      -name "f.ubar.com"


Generate a password for java keystore
-------------------------------------

Use same process as generating the key phrase, store the JKS password in jkspass.txt


Convert PKCS12 file to java keystore
------------------------------------

    keytool -importkeystore \
            -deststorepass `cat jkspass.txt` \
            -destkeypass `cat jkspass.txt` \
            -destkeystore host.jceks \
            -deststoretype JCEKS \
            -srckeystore host.p12 \
            -srcstoretype PKCS12 \
            -srcstorepass `cat p12pass.txt` \
            -alias f.ubar.com


View the keystore content
-------------------------

    keytool -list \
            -keystore host.jceks \
            -storetype JCEKS \
            -storepass `cat jkspass.txt`


Install on nginx
----------------

__TODO__


Install keystore on Tomcat
--------------------------

__TODO__


