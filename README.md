
# SNO - Single Node OpenShift för laborationer

Vi installerar detta på en fysisk maskin där vi kör CentOS 10 som operativsystem, 
med Cockpit är det enkelt att sätta upp den virtuella maskinen. Du behöver ha ett 
Red Hat-konto så att du kan logga in på [Hybrid Cloud Console](https://console.redhat.com).

I guiden går vi genom konfiguration av den virtuella servern, uppsättning av IPA 
server - om du inte redan har tillgång till en LDAP-lösning för authenticering av 
användarna, installationen av OpenShift och sedan konfigurering av limits, så eleverna 
inte startar upp alldeles för mycket och sänker maskinen.

Hittar du några fel, eller har förbättringsförslag får du gärna meddela oss eller 
göra en pull request! 

~ Andres, Martin och Jonas

## Virtual Machine

Vår virtuella maskin får bra med resurser.

|      |          |
|------|----------|
| RAM  | 256 GiB  |
| DISK | 256 GiB  |
| CPU  | 32 cores |

## Installera OCP med Assisted Installer

==TODO:== Hur vi installerar med Assisted.

## Installera IPA

Skapa en VM, I detta fallet kör vi CentOS Stream 10, och installera Free-IPA på den:

```sudo dnf install freeipa-server, ipa-server-dns, freeipa-server-trust-ad, chrony, bind```

Sätt korrekt hostname på servern
```hostnamectl set ipa.cloud2.se```

och lägg till den i ```/etc/hosts```
```135.181.173.242 ipa.cloud2.se ipa```

Kör
```sudo ipa-server-install ```
och följ instruktionerna. Vi vill installera IPA med en intern DNS-server

| hostname=ipa.cloud2.se |
| domain=cloud2.se | 
| realm=CLOUD2.SE setup-dns --allow-zone-overlap --auto-forwarders --auto-reverse

## Firewalld och äventyr med tyska polisen

Vi behöver även lite brandväggsregler för att släppa ut IPA.
Det finns färdiga services i Firewalld för IPA som man kan använda, men eftersom denna servern är tillgänglig publikt så behöver vi låsa ner brandväggen så att den bara tillåter anslutningar från vårt OpenShift-kluster, annars blir tyska polisen arga på Jonas.

Detta löser vi enklast med en liten regel i vår Firewalld-zon ```/etc/firewalld/zones/public.xml```

```
<?xml version="1.0" encoding="utf-8"?>
<zone>
  <short>Public</short>
  <description>For use in public areas. You do not trust the other computers on networks to not harm your computer. Only selected incoming connections are accepted.</description>
  <service name="ssh"/>
  <service name="dhcpv6-client"/>
  <service name="cockpit"/>
  <service name="freeipa-replication"/>
  <service name="dns"/>
  <service name="ntp"/>
  <service name="freeipa-4"/>
  <rule family="ipv4">
    <service name="dns"/>
    <drop/>
  </rule>
  <rule family="ipv4">
    <source address="135.181.173.243"/>
    <service name="dns"/>
    <accept/>
  </rule>
  <forward/>
</zone>
```
Sen laddar vi om brandväggen med ```sudo firewall-cmd --reload```

I samma veva löste vi även lite problem som vi hade på grund av att servern är publikt tillgänglig. Diverse botar, hackers och dylikt försökte hela tiden logga in som admin på servern, vilket efter 6 felaktiga försök låser kontot en stund. 

Därför installerar vi **Fail2Ban**, dock finns den inte i standardrepot för Centos Stream 10 så först får vi installera **EPEL**(Extra Packages for Enterprise Linux) först. 

```
sudo dnf install epel-release.noarch 
sudo dnf install fail2ban
sudo systemctl enable fail2ban && sudo systemctl start fail2ban
```



## Skapa DNS-records i IPA DNS

I zonen ```cloud2.se.``` behöver vi se till att följande DNS-records existerar. En del av dessa bör skapas automatiskt under installationsprocessen. 

| Record name | Record Type | Data |
| -------- | -------- | -------- |
| @   | NS    | ipa.cloud2.se.  |
| _kerberos | TXT | "CLOUD2.SE" |
| | URI | 0 100 "krb5srv:m:tcp:ipa.cloud2.se." |
| | URI | 0 100 "krb5srv:m:udp:ipa.cloud2.se." |
| _kerberos-master._tcp | SRV | 0 100 88 ipa.cloud2.se. |
| _kerberos-master._udp | SRV | 0 100 88 ipa.cloud2.se. |
| _kerberos._tcp | SRV | 0 100 88 ipa.cloud2.se. |
| _kerberos._udp | SRV | 0 100 88 ipa.cloud2.se. |
| _kpasswd | URI | 0 100 "krb5srv:m:tcp:ipa.cloud2.se." |
| | URI | 0 100 "krb5srv:m:udp:ipa.cloud2.se." |
| _kpasswd._tcp | SRV | 0 100 464 ipa.cloud2.se. |
| _kpasswd._udp | SRV | 0 100 464 ipa.cloud2.se. |
| ldap._tcp | SRV | 0 100 389 ipa.cloud2.se. |
| ipa | A | 135.181.173.242 |
| ipa-ca | A | 135.181.173.242 | 

_

## Anslut till IPA


## Synka användare från IPA till OCP

## Skapa project

```shell
oc new-project grupp1
oc new-project grupp2
oc new-project grupp3
oc new-project grupp4
oc new-project grupp5
```

## Sätt ResourceQuota och begränsningar

Vi sätter `ResourceQuota` på alla projekt:

```shell
oc apply -f - <<EOF
apiVersion: v1
kind: ResourceQuota
metadata:
  name: project-quota
  namespace: [PROJECT NAME HERE]
spec:
  hard:
    limits.cpu: "8"
    limits.memory: 16Gi
    persistentvolumeclaims: "10"
    pods: "20"
    requests.cpu: "4"
    requests.memory: 8Gi
EOF
```

Lägg till grupp så de kan komma åt sitt project:

```
oc adm policy add-role-to-group edit [GROUPNAME] -n [PROJECTNAME]
```

Gruppen *admin* i IPA skall ha admin-rättigheter i projects:

```
oc adm policy add-role-to-group admin openshift-admins -n [PROJECTNAME]
```

Ta bort möjligheten för användare att skapa nya project:

```
oc patch clusterrolebinding.rbac self-provisioners -p '{"subjects": null}'
oc patch clusterrolebinding.rbac self-provisioners -p '{ "metadata": { "annotations": { "rbac.authorization.kubernetes.io/autoupdate": "false" } } }'
```

Kontrollera att det blir rätt:

```
oc describe clusterrolebinding.rbac self-provisioners
``` 

## Sätt upp LVM storage

Installera LVM-operator:

```yaml
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  labels:
    operators.coreos.com/lvms-operator.openshift-lvm-storage: ""
  name: lvms-operator
  namespace: openshift-lvm-storage
spec:
  channel: stable-4.20
  installPlanApproval: Automatic
  name: lvms-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
  startingCSV: lvms-operator.v4.20.0
```

Skapa `StorageClass`:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  description: Provides RWO and RWOP Filesystem & Block volumes
  storageclass.kubernetes.io/is-default-class: "true"
  name: lvms-vg1
parameters:
  csi.storage.k8s.io/fstype: xfs
  topolvm.io/device-class: vg1
provisioner: topolvm.io
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
```




## Installera ArgoCD Operator

```shell
cat <<EOF | oc apply -f -
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: openshift-gitops-operator
  namespace: openshift-operators
spec:
  channel: latest
  installPlanApproval: Automatic
  name: openshift-gitops-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
EOF
```

Hitta URL och lösenord för *admin* :
```
URL=$(oc get route openshift-gitops-server -n openshift-gitops -o jsonpath='{.spec.host}')
PASS=$(oc get secret openshift-gitops-cluster -n openshift-gitops -o jsonpath='{.data.admin\.password}' | base64 -d)
```


