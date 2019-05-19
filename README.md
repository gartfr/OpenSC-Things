# OpenSC-Things

## Format the HSM
pkcs15-init -E

## Recreate Structure and Security PIN
pkcs15-init -C --pin 11111111 --puk 22222222 -p pkcs15+onepin

## Check it
pkcs15-tool --list-keys

## Maybe if not a new token, delete the previous Private Key and more
pkcs11-tool -l --delete-object --type privkey --id XX
(to be documented a bit more)

## Generate a new Private Key on the token
pkcs15-init --generate-key rsa/2048 --auth-id 01

## Identify you Private Key ID and Serial (to work with OpenSSL)
p11tool --list-all pkcs11:model=PKCS%2315
(model=ePass2003)

(extract from the result)
pkcs11:model=PKCS%2315;manufacturer=EnterSafe;serial=0325384116270713;token=User%20PIN%20%28YOYO%29;id=%16%b5%2c%a5%de%84%9b%40%ef%92%21%8a%58%54%2f%0f%a4%e6%82%52;object=Private%20Key;type=public

(extract useful information from above)
pkcs11:serial=0325384116270713;id=%16%b5%2c%a5%de%84%9b%40%ef%92%21%8a%58%54%2f%0f%a4%e6%82%52;

## OpenSSL, Generate a CSR from your Private Key whithout exporting the Private Key
engine dynamic -pre SO_PATH:/usr/lib/x86_64-linux-gnu/engines-1.1/pkcs11.so -pre ID:pkcs11 -pre LIST_ADD:1 -pre LOAD -pre MODULE_PATH:/usr/lib/x86_64-linux-gnu/pkcs11/opensc-pkcs11.so
(working on Ubuntu 18.04)

then, 

req -engine pkcs11 -new -key 'pkcs11:serial=0325384116270713;id=%16%b5%2c%a5%de%84%9b%40%ef%92%21%8a%58%54%2f%0f%a4%e6%82%52;' -keyform engine -out /home/gart/smartcard.pem.csr -text

## Give this CSR to your CA, you will have a Certificate as a result, import the Certificate to the token
pkcs15-init --store-certificate smartcard.pem.crt --auth-id 01 --format pem

## Extract the SSH Key from Token
pkcs15-tool --read-ssh-key 16b52ca5de849b40ef92218a58542f0fa4e68252

Will give you a result to be stored to your /.ssh/authorized_keys

ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDw.........WWXSS Private Key

## You can use it to connect your prefered server
ssh -I /usr/lib/x86_64-linux-gnu/onepin-opensc-pkcs11.so joe@server.prefered.com
