# BASH Common

```bash
#!/bin/bash
MACHINE_ID=""
CURL_RETVAL=0
CURL_HTTP_CODE=""
TTY_BOLD="\e[1m"
TTY_GREEN="\e[92m"
TTY_AMBER="\e[93;5m"
TTY_ODL_YELLOW="\e[93m"
TTY_ODL_RED="\e[31m"
TTY_RED="\e[101;97m"
TTY_BLUE="\e[94m"
TTY_HIDDEN="\e[8m"
TTY_REVERSE="\e[7m"
TTY_NORMAL="\e[0m"

#${var#*SubStr}  # drops substring from start of string up to first occurrence of `SubStr`
#${var##*SubStr} # drops substring from start of string up to last occurrence of `SubStr`
#${var%SubStr*}  # drops substring from last occurrence of `SubStr` to end of string
#${var%%SubStr*} # drops substring from first occurrence of `SubStr` to end of string

mk_tmp_file() {
 local TMP=$(mktemp -q -p ${CLI_TMP} ${cli_name}.$$.XXXXXXXX)
 eval $1=${TMP}
}

rm_tmp_file() {
 if [ -f "${1}" ] ; then
  rm -f "${1}" &> /dev/null
 fi
}

set_config() {
 local RETVAL=0
 grep "${1^^}=" ~/.${cli_name,,} &> /dev/null
 RETVAL=$?
 if [ ${RETVAL} -eq 0 ] ; then
   sed -i "/${1^^}=/ s\.*\\${1^^}=${2}\\" ~/.${cli_name,,}
   RETVAL=$?
 else
   sed -i "\$ a ${1^^}=${2}\\" ~/.${cli_name,,}
   RETVAL=$?
 fi
 return ${RETVAL}
}

delete_config() {
 local RETVAL=0
 grep "${1^^}=" ~/.${cli_name,,} &> /dev/null
 RETVAL=$?
 if [ ${RETVAL} -eq 0 ] ; then
   sed -i "/${1^^}=/ d" ~/.${cli_name,,}
   RETVAL=$?
 fi
 return ${RETVAL}
}

show_config() {
 local RETVAL=0
 grep "${1^^}=" ~/.${cli_name,,} &> /dev/null
 RETVAL=$?
 if [ ${RETVAL} -eq 0 ] ; then
   grep "${1^^}=" ~/.${cli_name,,}|sed -e "s\\^${1^^}=\\\\"
   RETVAL=$?
 fi
 return ${RETVAL}
}

get_config() {
 local RETVAL=0
 grep "${1^^}=" ~/.${cli_name,,} &> /dev/null
 RETVAL=$?
 if [ ${RETVAL} -eq 0 ] ; then
   CONFIG_VALUE=$(grep "${1^^}=" ~/.${cli_name,,}|sed -e "s\\^${1^^}=\\\\")
   RETVAL=$?
 fi
 return ${RETVAL}
}

show_config_all() {
 grep -E "^ODL_([A-Z0-9_-]*)=" ~/.${cli_name,,}|sed -e "s\\^ODL_\\\\"
 RETVAL=$?
 return ${RETVAL}
}

set_machine_id() {
 if [ -f /etc/machine-id ] ; then
  MACHINE_ID=$(cat /etc/machine-id)
 elif [ -f /var/lib/dbus/machine-id ] ; then
  MACHINE_ID=$(cat /var/lib/dbus/machine-id)
 else
  MACHINE_ID=$(xxd -l16 -ps /dev/urandom)
 fi
 set_config "MACHINE_ID" "${MACHINE_ID}"
 RETVAL=$?
 return ${RETVAL}
}

get_machine_id() {
 get_config "MACHINE_ID" MACHINE_ID
 MACHINE_ID="$CONFIG_VALUE"
}

encrypt() {
 if [[ ${OPENSSL_VERSION,,} =~ "openssl1.1." ]] ; then
   CONFIG_VALUE=$(echo "${1}" | openssl enc -e -aes-256-cbc -pbkdf2 -iter 2174 -a -k ${MACHINE_ID} )
 else
   CONFIG_VALUE=$(echo "${1}" | openssl enc -e -aes-256-cbc -a -k ${MACHINE_ID} )
 fi
 return $?
}

decrypt() {
 if [[ ${OPENSSL_VERSION,,} =~ "openssl1.1." ]] ; then
   CONFIG_VALUE=$(echo "${1}" | openssl enc -d -aes-256-cbc -pbkdf2 -iter 2174 -a -k ${MACHINE_ID} )
 else
   CONFIG_VALUE=$(echo "${1}" | openssl enc -d -aes-256-cbc -a -k ${MACHINE_ID} )
 fi
 return $?
}

set_password() {
 local RETVAL=0
 encrypt "${1}"
 RETVAL=$?
 set_config "ODL_PASSWORD" "${CONFIG_VALUE}"
 CONFIG_VALUE=""
 RETVAL=$?
 return ${RETVAL}
}

set_jolokia_password() {
 local RETVAL=0
 encrypt "${1}"
 RETVAL=$?
 set_config "JOLOKIA_PASSWORD" "${CONFIG_VALUE}"
 CONFIG_VALUE=""
 RETVAL=$?
 return ${RETVAL}
}

get_password() {
 local RETVAL=0
 get_config "ODL_PASSWORD"
 decrypt "${CONFIG_VALUE}"
 RETVAL=$?
 return ${RETVAL}
}

get_jolokia_password() {
 local RETVAL=0
 get_config "JOLOKIA_PASSWORD"
 decrypt "${CONFIG_VALUE}"
 RETVAL=$?
 return ${RETVAL}
}

# 1 var response output file
# 2 http method
# 3 URL path
# 4 body file
# 5 content-type value
# 6 accept value
# 7 port override
http_curl() {
 local CURL_REQUEST="--location --request ${2^^}"
 local CURL_PATH="${3}"
 local CURL_RESPONSE="${1}"
 local CURL_CONTENT_TYPE="${5}"
 local CURL_ACCEPT="${6}"
 local HTTP_HEADERS=""
 get_password
 local PASSWORD="${CONFIG_VALUE}"

 CURL_HTTP_CODE=$(/usr/bin/curl -s \
  -o ${CURL_RESPONSE} \
  -w '%{http_code}' \
  --insecure \
  --connect-timeout ${ODL_TIMEOUT} \
  --max-time ${ODL_MAXTIME} \
  --user-agent "${USER_AGENT}" \
  -u "${ODL_USER}:${PASSWORD}" \
  -H "Cache-control: no-cache" \
  -H "Accept: ${CURL_ACCEPT}" \
  -H "Content-Type: ${CURL_CONTENT_TYPE}" \
  -d @${4} \
  ${CURL_REQUEST} \
  "${ODL_PROTOCOL,,}://${ODL_HOST}:${ODL_PORT}${CURL_PATH}" )
  CURL_RETVAL=$?
  if [ "${CURL_HTTP_CODE:0:1}" == "4" ] || [ "${CURL_HTTP_CODE:0:1}" == "5" ] ; then
    CURL_RETVAL=${CURL_HTTP_CODE}
    local CURL_RESTCONF=""
    case ${CURL_HTTP_CODE} in
      400)         CURL_RESTCONF="(response) too-big, malformed-message, missing/bad attribute or element" ;;
      401)         CURL_RESTCONF="access denied" ;;
      403)         CURL_RESTCONF="forbidden" ;;
      404)         CURL_RESTCONF="not-found" ;;
      406)         CURL_RESTCONF="invalid-value" ;;
      405)         CURL_RESTCONF="operation-not-supported" ;;
      409)         CURL_RESTCONF="data-exists, data-missing, in-use or lock or resource denied" ;;
      412)         CURL_RESTCONF="operation-failed" ;;
      413)         CURL_RESTCONF="(request) too-big" ;;
      4[0-9][0-9]) CURL_RESTCONF="bad request" ;;
      500)         CURL_RESTCONF="operation-failed,partial-option or rollback-failed" ;;
      501)         CURL_RESTCONF="operation-not-supported" ;;
      5[0-9][0-9]) CURL_RESTCONF="bad request" ;;
    esac
    [[ -n "${CURL_RESTCONF}" ]] && echo "Error: ${CURL_RESTCONF} [${CURL_HTTP_CODE}]"
  fi
  [[ ${CURL_RETVAL} -eq 18 ]] && CURL_RETVAL=0
}

# 1 var response output file
# 2 http method
# 3 URL path
# 4 body file
# 5 content-type value
# 6 accept value
# 7 port override
jolokia_curl() {
 local CURL_REQUEST="--location --request ${2^^}"
 local CURL_PATH="${3}"
 local CURL_RESPONSE="${1}"
 local CURL_CONTENT_TYPE="${5}"
 local CURL_ACCEPT="${6}"
 local HTTP_HEADERS=""
 get_jolokia_password
 local PASSWORD="${CONFIG_VALUE}"

 CURL_HTTP_CODE=$(/usr/bin/curl -s \
  -o ${CURL_RESPONSE} \
  -w '%{http_code}' \
  --insecure \
  --connect-timeout ${ODL_TIMEOUT} \
  --max-time ${ODL_MAXTIME} \
  --user-agent "${USER_AGENT}" \
  -u "${JOLOKIA_USER}:${PASSWORD}" \
  -H "Cache-control: no-cache" \
  -H "Accept: ${CURL_ACCEPT}" \
  -H "Content-Type: ${CURL_CONTENT_TYPE}" \
  -d @${4} \
  ${CURL_REQUEST} \
  "${ODL_PROTOCOL,,}://${ODL_HOST}:${ODL_PORT}${CURL_PATH}" )
  CURL_RETVAL=$?
  [[ ${CURL_RETVAL} -eq 18 ]] && CURL_RETVAL=0
}

get_yang_tools_version() {
  local RETVAL=0
  get_password
  local PASSWORD="${CONFIG_VALUE}"
  local VERSION=$(/usr/bin/curl -s \
                    --insecure \
                    --connect-timeout ${ODL_TIMEOUT} \
                    --max-time ${ODL_MAXTIME} \
                    --user-agent "${USER_AGENT}" \
                    -u "${ODL_USER}:${PASSWORD}" \
                    -H "Cache-control: no-cache" \
                    -H "Accept: ${CURL_ACCEPT}" \
                    -H "Content-Type: ${CURL_CONTENT_TYPE}" \
                    --location --request GET \
                    "${ODL_PROTOCOL,,}://${ODL_HOST}:${ODL_PORT}/jolokia/read/org.apache.karaf:type=bundle,name=root" | \
                 jq -r '.value.Bundles | to_entries[] | select(.value."Symbolic Name" == "org.opendaylight.yangtools.concepts") | .value.Version'
                 )
  RETVAL=$?
  if [ ${RETVAL} -eq 0 ] ; then
    set_config YANG_TOOLS "${VERSION}"
    YANG_TOOLS="${VERSION}"
  fi
}

show_banner()
{
 if [ -f "${CLI_LIB}/banner.${cli_name,,}" ] ; then
  [[ ${CLI_INTERACTIVE} -eq 0 ]] && echo -n -e "${TTY_ODL_YELLOW}"
  cat ${CLI_LIB}/banner.${cli_name,,}
  echo -e "                               ${TTY_ODL_YELLOW}OpenDaylight CLI${TTY_ODL_YELLOW}"
  echo " "
  [[ ${CLI_INTERACTIVE} -eq 0 ]] && echo -n -e "${TTY_NORMAL}"
 fi
}

show_version() {
  echo -e "                                 ${TTY_GREEN}${cli_name} ${TTY_LIGHT_BLUE}${CLI_VERSION}${TTY_NORMAL}\n"
}

show_copyright() {
 [[ ${CLI_INTERACTIVE} -eq 0 ]] && echo -n -e "${TTY_ODL_RED}"
 echo "               Cowdrey Consulting UK Ltd <https://www.cowdrey.co.uk/>"
 [[ ${CLI_INTERACTIVE} -eq 0 ]] && echo -n -e "${TTY_ODL_YELLOW}"
 echo "               OpenDaylight Project <https://www.opendaylight.org/>"
 echo "                    License Eclipse Public License (EPL-1.0)"
 echo "    <https://www.opendaylight.org/technical-community/ip-policy/licensing>"
 echo "    <https://www.eclipse.org/legal/eplfaq.php#PARTIESEPL>"
 echo " "
 echo "       This is free software; you are free to change and redistribute it."
 echo "            There is NO WARRANTY, to the extent permitted by law."
}

install_jolokia() {
 echo "${cli_name} - warning Jolokia Monitoring Support is not deployed to SDNc."
 echo "Please connect to Karaf console client and install, example below:"
 echo " "
 echo "\$ ssh -p 8101 -o UserKnownHostsFile=/dev/null -o StrictHostKeyChecking=no karaf@localhost"
 echo "opendaylight-user@karaf> feature:install odl-jolokia"
 echo "opendaylight-user@karaf> feature:install jolokia"
 echo "opendaylight-user@karaf> feature:list | grep jolokia"
 echo "odl-jolokia                                                     │ 1.13.2           │ x        │ Started     │ odl-extras-1.13.2                                               │ Jolokia JMX/HTTP bridge"   echo "jolokia                                                         │ 1.6.1            │ x        │ Started     │ standard-4.2.6                                                  │ Jolokia monitoring support"
 echo "opendaylight-user@karaf> <CTRL>D"
 echo "\$ exit"
 echo " "
}

cli_help() {
  local RETVAL=0
  if [ ${CLI_INTERACTIVE} -eq 0 ] ; then
    echo -e "Usage: ${TTY_BOLD}[cmd]${TTY_NORMAL}"
    echo -e "${TTY_BOLD}Commands${TTY_NORMAL}:"
  else
    echo "Usage: ${cli_name} [cmd]"
    echo "Commands:"
  fi
  find ${CLI_LIB}/ -type f -executable -printf "\t%f\n"|grep -v common
  if [ ${CLI_INTERACTIVE} -eq 0 ] ; then
    echo -e "\tquit\t Exit the CLI"
  fi
  RETVAL=$?
  return ${RETVAL}
}

aaa_help_list() {
  local RETVAL=0
  if [ ${CLI_INTERACTIVE} -eq 0 ] ; then
    echo -e "Usage: ${TTY_BOLD}aaa [key] [action] [options]${TTY_NORMAL}"
    echo -e "${TTY_BOLD}Commands${TTY_NORMAL}:"
  else
    echo "Usage: ${cli_name} aaa [key] [action] [options]"
    echo "Commands:"
  fi
  echo "   domains add        <<domain-name>>"
  echo "                      [--description <<text>>]"
  echo "                      [--enable]"
  echo "   domains disable    <<domain-name>>"
  echo "   domains enable     <<domain-name>>"
  echo "   doamins list"
  echo "   domains delete     <<domain-name>>"
  echo " "
  echo "   roles add          <<role-name>>"
  echo "                      [--domain <<domain-name>>]"
  echo "   roles list         [--domain <<domain-name>>]"
  echo "   roles delete       <<role-name>>"
  echo "                      [--domain <<domain-name>>]"
  echo " "
  echo "   users add          <<username>>"
  echo "                      [--domain <<domain-name>>]"
  echo "                      [--description <<text>>]"
  echo "                      [--enable]"
  echo "                      [--email <<text>>]"
  echo "                      [--password <<text>>]"
  echo "   users delete       <<username"
  echo "                      [--domain <<domain-name>>]"
  echo "   users disable      <<username>>"
  echo "                      [--domain <<domain-name>>]"
  echo "   users enable       <<username>>"
  echo "                      [--domain <<domain-name>>]"
  echo "   users list         [[-domain <<domain-name>>]"
  echo "   users password     <<username>>"
  echo "                      --password <<text>>"
  echo "                      [--domain <<domain-name>>]"
  echo " "
  echo "   policies add       <<url-resource>>"
  echo "                      <<role-name>>"
  echo "                      [--index <<index-value>> ]"
  echo "                      [--get] [--post] [--put] [--patch] [--delete]"
  echo "                      [--insert] [--append] [--line <<line-number>>]"
  echo "                      [--domain <<domain-name>>]"
  echo "   policies list      [--role <<role-name>>]"
  echo "                      [--domain <<domain-name>>]"
  echo "   policies delete    <<url-resourcee>>"
  echo "                      [--role <<role-name>>]"
  echo "                      [--domain <<domain-name>>]"
  echo " "
  if [ ${CLI_INTERACTIVE} -eq 0 ] ; then
    echo -e " {$TTY_BOLD}Attention\e[0m"
    echo -e " - when <<domain-name>> is not supplied, domain will default to {$TTY_BOLD}${ODL_AAA_DEFAULT_DOMAIN}${TTY_NORMAL}"
    echo -e " - the default domain {$TTY_BOLD}${ODL_AAA_DEFAULT_DOMAIN}\e[0m can not be deleted, nor can the default user {$TTY_BOLD}${ODL_AAA_DEFAULT_USER}\e[0m (on domain {$TTY_BOLD}${ODL_AAA_DEFAULT_DOMAIN}\e[0m)"
  else
    echo " Attention:"
    echo " - when <<domain-name>> is not supplied, domain will default to '${ODL_AAA_DEFAULT_DOMAIN}'"
    echo " - the default domain '${ODL_AAA_DEFAULT_DOMAIN}' can not be deleted, nor can the default user '${ODL_AAA_DEFAULT_USER}' (on domain '${ODL_AAA_DEFAULT_DOMAIN}')"
  fi
    echo " - insert/append new policies will be inserted before or appended after the supplied <<line-number>> - default is append"
    echo " - if no <<line-number>> supplied, append will be after exsting last policy and insert will be before last policy"
    echo " - URL resources should include all appropriate wildcards, i.e. /restconf/** , /rests/data/network-topology:network-topology/topology=topology-netconf/**"
  echo " "
  if [ ${CLI_INTERACTIVE} -eq 0 ] ; then
    echo -e "\e[31m\e[1mWarning${TTY_NORMAL}:"
    echo -e " - {$TTY_BOLD}ALL\e[0m deletes will automatically propagate where appropriate i.e. deleting a domain will also delete associated users and roles etc.; deleting a role will also delete associated policies"
  else
    echo " Warning:"
    echo " - 'ALL' deletes will automatically propagate where appropriate i.e. deleting a domain will also delete associated users and roles etc.; deleting a role will also delete associated policies"
  fi
  return ${RETVAL}
}

nodes_help_list() {
  local RETVAL=0
  if [ ${CLI_INTERACTIVE} -eq 0 ] ; then
    echo -e "Usage: ${TTY_BOLD}nodes [cmd]${TTY_NORMAL}"
    echo -e "${TTY_BOLD}Commands${TTY_NORMAL}:"
  else
    echo "Usage: ${cli_name} nodes [cmd]"
    echo "Commands:"
  fi
  echo "   aaa <<node-id>>                       Verify AAA access to specific node"
  echo "       [ --name                        ] username to attempt access with, if not specified defualt will be used"
  echo "       [ --password                    ] password to attempt access with, if not specififed default will be used"
  echo "   export                                Exported listed nodes"
  echo "        [ --encrypt                    ] encrypt passwords"
  echo "   list                                  List known nodes"
  echo "   status                                List connection status of known nodes"
  echo "        [ --summary                    ] List number of nodes connected, connecting, unable-to-connect in summary form"
  echo "   state  <<node-id>> | --all            ONF control-construct operational state"
  echo "       [  --yang-model <<model>>       ] YANG model, defaults to core-model-1-4"
  echo "       [  --exclude <<node-id>>        ] Exclude specified node, repeat as necessary"
  echo "       [  --extended                   ] Fetch LTPs and report status and size"
  echo "       [  --timeout <<milliseconds>>   ] Maximum wait (milliseconds, default 60000)"
  echo "   test <<node-id>> | --all              PING, SSH verify, NETCONF verify specified node or all known"
  echo "       [  --timeout <<seconds>>        ] Maximum wait (seconds)"
  echo "       [  --capabilities]              ] additional display reported NETCONF capabilities of the node(s)"
  echo "   mount <<node-id>>                     Mount network device/node"
  echo "          --host <<ip>>                  node IP or FQDN"
  echo "          --username <<text>>            node NETCONF username"
  echo "          --password <<text>>            node NETCONF password"
  echo "       [ --encrypted-password <<text>> ] use supplied encrypted password"
  echo "       [ --port <<port number>>        ] node NETCONF port (default: 830)"
  echo "       [ --cache <<dir>>               ] mount node schema cache directory"
  echo "       [ --cschema                     ] node reconnect on changed schema"
  echo "       [ --timeout <<milliseconds>>    ] node default request timeout"
  echo "       [ --tcp-only                    ] node limited to TCP only"
  echo "       [ --max-attempts <<attempts>>   ] maximum attempts of connecting to node, default 0/unlimited"
  echo "       [ --concurrent <<rpc-limit>>    ] maximum of RPC to send before response received, default 0/unlimited"
  echo "   ummount <<node-id>> | --all           Unmount network device/node or all"
  return ${RETVAL}
}

delete_help_list() {
  local RETVAL=0
  if [ ${CLI_INTERACTIVE} -eq 0 ] ; then
    echo -e "Usage: ${TTY_BOLD}delete [parameter]${TTY_NORMAL}"
    echo -e "${TTY_BOLD}Parameters${TTY_NORMAL}:"
  else
    echo "Usage: ${cli_name} delete [parameter]"
    echo "Parameters:"
  fi
  echo "   user/pass/protocol/host/port/timeout/maxtime can not be deleted"
  echo "   <<name>>         Delete custom parameter"
  return ${RETVAL}
}

query_help_list() {
  local RETVAL=0
  if [ ${CLI_INTERACTIVE} -eq 0 ] ; then
    echo -e "Usage: ${TTY_BOLD}query [method] [path]${TTY_NORMAL}"
    echo -e "${TTY_BOLD}Parameters${TTY_NORMAL}:"
  else
    echo "Usage: ${cli_name} query [method] [path]"
    echo "Parameters:"
  fi
  echo "   get '<<URL Path>>'                   RESTCONF Get URL path, do not include /rests/data prefix i.e."
  echo "                                        GET '/network-topology:network-topology/topology=topology-netconf?content=nonconfig&fields=node(node-id)'"
  return ${RETVAL}
}

set_help_list() {
  local RETVAL=0
  if [ ${CLI_INTERACTIVE} -eq 0 ] ; then
    echo -e "Usage: ${TTY_BOLD}set [parameter] [value]${TTY_NORMAL}"
    echo -e "${TTY_BOLD}Parameters${TTY_NORMAL}:"
  else
    echo "Usage: ${cli_name} set [parameter] [value]"
    echo "Parameters:"
  fi
  echo "   user      Set RESTCONF API username"
  echo "   pass      Set RESTCONF API password"
  echo "   protocol  Set RESTCONF API protocol"
  echo "   host      Set RESTCONF API host"
  echo "   port      Set RESTCONF API port number"
  echo "   timeout   Set RESTCONF API connection timeout (seconds)"
  echo "   maxtime   Set RESTCONF API max wait timeout (seconds)"
  echo "   <<name>>  Set custom parameter"
  echo "   juser     Set Jolokia username"
  echo "   jpass     Set Jolokia password"
  return ${RETVAL}
}

show_help_list() {
  local RETVAL=0
  if [ ${CLI_INTERACTIVE} -eq 0 ] ; then
    echo -e "Usage: ${TTY_BOLD}show [parameter] | --all\e[0m"
    echo -e "${TTY_BOLD}Parameters${TTY_NORMAL}:"
  else
    echo "Usage: ${cli_name} show [parameter] | --all"
    echo "Parameters:"
  fi
  echo "   user        Show RESTCONF API username"
  echo "   pass        Show RESTCONF API password"
  echo "   protocol    Show RESTCONF API protocol"
  echo "   host        Show RESTCONF API host"
  echo "   port        Show RESTCONF API port number"
  echo "   timeout     Show RESTCONF API connection timeout (seconds)"
  echo "   maxtime     Show RESTCONF API max wait timeout (seconds)"
  echo "   juser       Show Jolokia username"
  ehco "   jpass       Show Jolokia password"
  echo " Not included within --all"
  echo "   capability  Show RESTCONF API offered capabilities"
  echo "   <<name>>    Show customer parameter"
  return ${RETVAL}
}

sites_help_list() {
  local RETVAL=0
  if [ ${CLI_INTERACTIVE} -eq 0 ] ; then
    echo -e "Usage: ${TTY_BOLD}sites [cmd]${TTY_NORMAL}"
    echo -e "${TTY_BOLD}Commands${TTY_NORMAL}:"
  else
    echo "Usage: ${cli_name} sites [cmd]"
    echo "Commands:"
  fi
  echo "   list [svn-name]            List known sites vpn-name and nodes"
  return ${RETVAL}
}

streams_help_list() {
  local RETVAL=0
  if [ ${CLI_INTERACTIVE} -eq 0 ] ; then
    echo -e "Usage: ${TTY_BOLD}streams [cmd]${TTY_NORMAL}"
    echo -e "${TTY_BOLD}Commands${TTY_NORMAL}:"
  else
    echo "Usage: ${cli_name} streams [cmd]"
    echo "Commands:"
  fi
  echo "    bootstrap                  Used to intialize the RFC8040 Events/Notification streams upon SDNc startup"
  echo "                               note: should not be generally used but will not cause harm and can be used"
  echo "                                     to verify availability of capability"
  echo "    list                       List currently available RFC8040 Events/Notification streams"
  echo "         [ --json-only       ] limit to JSON encoded streams"
  echo "         [ --xml-only        ] limit to XML encoded streams"
  echo "    monitor <<stream>>         Interactively subscribe and display RFC8040 stream"
  echo "                               <<stream>> is stream location segment"
  echo "                               i.e. ietf-netconf-notifications:netconf-config-change"
  echo "         [ --json            ] JSON encoding, rather than default XML encoding"
  return ${RETVAL}
}

stats_help_list() {
  local RETVAL=0
  if [ ${CLI_INTERACTIVE} -eq 0 ] ; then
    echo -e "Usage: ${TTY_BOLD}stats [cmd]${TTY_NORMAL}"
    echo -e "${TTY_BOLD}Commands${TTY_NORMAL}:"
  else
    echo "Usage: ${cli_name} stats [cmd]"
    echo "Commands:"
  fi
  echo "   runtime   List JVM runtime"
  echo "   os        List Operating System"
  echo "   memory    List JVM memory usage"
  echo "   version   List ${cli_name} component versions"
  echo "   nodecount List Lumina Node Counter"
  return ${RETVAL}
}

```
