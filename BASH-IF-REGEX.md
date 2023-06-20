# BASH IF REGEX
```bash
if [[ ${OPENSSL_VERSION,,} =~ "openssl1.1." ]] ; then 
  CONFIG_VALUE=$(echo "${1}" | openssl enc -e -aes-256-cbc -pbkdf2 -iter 2174 -a -k ${MACHINE_ID} ) 
else 
  CONFIG_VALUE=$(echo "${1}" | openssl enc -e -aes-256-cbc -a -k ${MACHINE_ID} ) 
fi
```
