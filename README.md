# OpenSC-Things

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

###### OpenSSL, Generate a CSR
`engine dynamic -pre SO_PATH:/usr/lib/x86_64-linux-gnu/engines-1.1/pkcs11.so -pre ID:pkcs11 -pre LIST_ADD:1 -pre LOAD -pre MODULE_PATH:/usr/lib/x86_64-linux-gnu/pkcs11/opensc-pkcs11.so`
(working on Ubuntu 18.04)
This will initialize OpenSSL variable to access the device

then, 

`req -engine pkcs11 -new -key 'pkcs11:serial=0325384116270713;id=%16%b5%2c%a5%de%84%9b%40%ef%92%21%8a%58%54%2f%0f%a4%e6%82%52;' -keyform engine -out /home/gart/smartcard.pem.csr -text`
Generate the CSR

###### Store the certificate
`pkcs15-init --store-certificate smartcard.pem.crt --auth-id 01 --format pem`
Give this CSR to your CA, you will have a Certificate as a result, import the Certificate to the token

## Play with SSH auth
###### Extract the SSH Key from Token
`pkcs15-tool --read-ssh-key 16b52ca5de849b40ef92218a58542f0fa4e68252`

Will give you a result to be stored to your /.ssh/authorized_keys

`ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDw.........WWXSS Private Key`

###### You can use it to connect your prefered server
`ssh -I /usr/lib/x86_64-linux-gnu/onepin-opensc-pkcs11.so joe@server.prefered.com`


## Play with non Cetificate Objects
###### Add a Truecrypt or Keepass KeyFile
`pkcs11-tool --login --write-object key.tc --type data --id 2 --label 'key.tc' --private`

`pkcs11-tool --login --write-object keepass.key --type data --id 3 --label 'keepass.key' --private`

###### Read an Object to verify
`pkcs11-tool --login --read-object --type data --label 'keepass.key'`

###### Read an Object with no Label
`pkcs11-tool --login --read-object --type data --label ''`
