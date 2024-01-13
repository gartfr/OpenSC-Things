# OpenSC Things

## Initialize the token
###### Format the HSM
`pkcs15-init -E`

Will reset and format the PKCS hardware token for use with opensc.

###### Recreate Structure and Security PIN
`pkcs15-init -C --pin 11111111 --puk 22222222 -p pkcs15+onepin`

Will assign a pin to unlock the token and create a pkcs structure

###### Check it
`pkcs15-tool --list-keys`

To display the token information and public key information

###### Maybe if not a new token, delete the previous Private Key and more
`pkcs11-tool -l --delete-object --type privkey --id XX`

(to be documented a bit more, not tested, somtimes buggy on some devices)

###### Generate a new Private Key on the token
`pkcs15-init --generate-key rsa/2048 --auth-id 01`

Generate a unique private key on the token. It will never go out of the token, and can only be unlocked by pin.

## Play with OpenSSL 
This will permit us to have a corresponding certificate based upon our private key

###### Identify your Private Key ID and Serial (to work with OpenSSL)
`p11tool --list-all pkcs11:model=PKCS%2315`
(model=ePass2003)

(extract from the result)
`pkcs11:model=PKCS%2315;manufacturer=EnterSafe;serial=0325384116270713;token=User%20PIN%20%28YOYO%29;id=%16%b5%2c%a5%de%84%9b%40%ef%92%21%8a%58%54%2f%0f%a4%e6%82%52;object=Private%20Key;type=public`

(extract useful information from above)
`pkcs11:serial=0325384116270713;id=%16%b5%2c%a5%de%84%9b%40%ef%92%21%8a%58%54%2f%0f%a4%e6%82%52;`

###### Customize OpenSSL to use PKCS11 token
create a config file :
enginepkcs11.conf 
```
    openssl_conf = openssl_init

    [openssl_init]
    engines = engine_section

    [engine_section]
    pkcs11 = pkcs11_section

    [pkcs11_section]
    engine_id = pkcs11
#    dynamic_path is not required if you have installed
#    the appropriate pkcs11 engines to your openssl directory
#    dynamic_path = /path/to/engine_pkcs11.{so|dylib}
    MODULE_PATH = /usr/lib/aarch64-linux-gnu/pkcs11/opensc-pkcs11.so
#    it is not recommended to use "debug" for production use
#    INIT_ARGS = connector=http://127.0.0.1:12345 debug
#    init = 0

[ req ]
distinguished_name = req_dn
string_mask = utf8only
utf8 = yes

[ req_dn ]
commonName = Common Name
```

###### OpenSSL, Generate a CSR
<!-- `engine dynamic -pre SO_PATH:/usr/lib/x86_64-linux-gnu/engines-1.1/pkcs11.so -pre ID:pkcs11 -pre LIST_ADD:1 -pre LOAD -pre MODULE_PATH:/usr/lib/x86_64-linux-gnu/pkcs11/opensc-pkcs11.so`
(working on Ubuntu 18.04)

This will initialize OpenSSL variable to access the device

then, 

`req -engine pkcs11 -new -key 'pkcs11:serial=0325384116270713;id=%16%b5%2c%a5%de%84%9b%40%ef%92%21%8a%58%54%2f%0f%a4%e6%82%52;' -keyform engine -out /home/gart/smartcard.pem.csr -text` -->

`OPENSSL_CONF=enginepkcs11.conf openssl req -new -engine pkcs11 -key 'pkcs11:serial=0325384116270713;id=%16%B5%2C%A5%DE%84%9B%40%EF%92%21%8A%58%54%2F%0F%A4%E6%82%52;' -keyform engine -out ~/smartcard.pem.csr -text`

Generate the CSR.
Give this CSR to your CA, you will have a Certificate as a result, import the Certificate to the token

###### Store the certificate
`pkcs15-init --store-certificate smartcard.pem.crt --auth-id 01 --format pem`

###### Update a certificate
`pkcs15-init -X smartcard.crt --format pem --id 00c3b2d268a75fad5b23f5da5198bac956ef311a -a 01`


## Play with SSH auth
###### Extract the SSH Key from Token
`pkcs15-tool --read-ssh-key 16b52ca5de849b40ef92218a58542f0fa4e68252`

Will give you a result to be stored to your /.ssh/authorized_keys

`ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDw.........WWXSS Private Key`

###### You can use it to connect your prefered server
`ssh -I /usr/lib/x86_64-linux-gnu/onepin-opensc-pkcs11.so joe@server.prefered.com`

The parameter `-I` permits to play with hardware token.

###### Add the feature for users forever
`sudo vim /etc/ssh/ssh_config.d/pkcs-provider.conf`
```
PKCS11Provider /usr/lib/aarch64-linux-gnu/pkcs11/opensc-pkcs11.so
```
Adapt the path regarding your Linux distrib


## Play with non Certificate Objects
###### Add a Truecrypt or Keepass KeyFile
```
pkcs11-tool --login --write-object key.tc --type data --id 2 --label 'key.tc' --private`
pkcs11-tool --login --write-object keepass.key --type data --id 3 --label 'keepass.key' --private
```

Those 2 lines are examples to store files on the token (here Truecrypt/Veracrypt and Keepass examples)
Those files can be access on the token through PKCS11 interface through the software.
It will ask the pin code to unlock access to the files.


###### Read an Object to verify
`pkcs11-tool --login --read-object --type data --label 'keepass.key'`

###### Read an Object with no Label
`pkcs11-tool --login --read-object --type data --label ''`
