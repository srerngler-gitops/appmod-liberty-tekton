git clone https://github.com/ibm-cloud-architecture/appmod-liberty-tekton.git
cd appmod-liberty-tekton

#####
## setup the db2 into the db2 namespace and populate the orderdb
####
Note - On IBM Cloud, db2 requires norootsquash
oc project kube-system
#oc create secret docker-registry cpregistrysecret\
    --docker-server=cp.icr.io/cp/cpd\
    --docker-username=cp\
    --docker-password=entitlement_key\
    --docker-email=your_email_address
oc create secret docker-registry cpregistrysecret --docker-username=cp --docker-password=eyJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJJQk0gTWFya2V0cGxhY2UiLCJpYXQiOjE1NzU4OTMzMTMsImp0aSI6ImQzNGVjMzg5ZDkzYTQ0NWZhMzZmNDVmNzYzZmY5MWMwIn0.H45JE9YqidIcXPWC7xxfZQ05WD9tSFEZtcVLArzZ1ck --docker-server=cp.icr.io/cp/cpd 

oc apply -f ./norootsquash.yaml


1. create the db2 namespace
oc new-project db2
oc create secret docker-registry ibm-entitlement-key --docker-username=cp --docker-password=eyJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJJQk0gTWFya2V0cGxhY2UiLCJpYXQiOjE1NzU4OTMzMTMsImp0aSI6ImQzNGVjMzg5ZDkzYTQ0NWZhMzZmNDVmNzYzZmY5MWMwIn0.H45JE9YqidIcXPWC7xxfZQ05WD9tSFEZtcVLArzZ1ck --docker-server=cp.icr.io --namespace=db2

2. Install the ibm db2 operator into the db2 namespace
oc create -f - <<EOF
apiVersion: operators.coreos.com/v1alpha1
kind: CatalogSource
metadata:
  name: ibm-operator-catalog
  namespace: openshift-marketplace
spec:
  displayName: "IBM Operator Catalog" 
  publisher: IBM
  sourceType: grpc
  image: icr.io/cpopen/ibm-operator-catalog
  updateStrategy:
    registryPoll:
      interval: 45m
EOF

3. Apply the db2u-scc
oc create -f - <<EOF
kind: SecurityContextConstraints
apiVersion: v1
apiGroup: security.openshift.io
metadata:
    name: db2u-scc
allowHostDirVolumePlugin: false
allowHostIPC: false
allowHostNetwork: false
allowHostPID: false
allowHostPorts: false
# privileged container is only needed for the init container that sets the Db2 kernel parameters
allowPrivilegedContainer: true
allowedCapabilities:
- "SYS_RESOURCE"
- "IPC_OWNER"
- "SYS_NICE"
- "CHOWN"
- "DAC_OVERRIDE"
- "FSETID"
- "FOWNER"
- "SETGID"
- "SETUID"
- "SETFCAP"
- "SETPCAP"
- "SYS_CHROOT"
- "KILL"
- "AUDIT_WRITE"
priority: 10
runAsUser:
    type: RunAsAny
seLinuxContext:
    type: MustRunAs
fsGroup:
    type: RunAsAny
supplementalGroups:
    type: RunAsAny
version: v1
EOF

4. Install the db2 instance
oc create -f - <<EOF
apiVersion: db2u.databases.ibm.com/v1
kind: Db2uCluster
metadata:
  name: db2
  labels:
    app.kubernetes.io/instance: db2u-operator
    app.kubernetes.io/managed-by: Db2U-Team
    app.kubernetes.io/name: db2u-operator
  namespace: db2
spec:
  size: 1
  license:
    accept: true
    value: >-
      W0xpY2Vuc2VDZXJ0aWZpY2F0ZV0KQ2hlY2tTdW09NzYwMEZGQTMyMTI1NzQyMzAxRTk2OUY1NkQ5NEZFNkUKVGltZVN0YW1wPTE1NTc0MTcxNTQKUGFzc3dvcmRWZXJzaW9uPTQKVmVuZG9yTmFtZT1JQk0gVG9yb250byBMYWIKVmVuZG9yUGFzc3dvcmQ9N3Y4cDRmcTJkdGZwYwpWZW5kb3JJRD01ZmJlZTBlZTZmZWIuMDIuMDkuMTUuMGYuNDguMDAuMDAuMDAKUHJvZHVjdE5hbWU9REIyIEFkdmFuY2VkIEVkaXRpb24KUHJvZHVjdElEPTE0MDQKUHJvZHVjdFZlcnNpb249MTEuNQpQcm9kdWN0UGFzc3dvcmQ9Ynd1NXppdXd4cXN6bmd3NWFmNHUyYjcyClByb2R1Y3RBbm5vdGF0aW9uPTEyNyAxNDMgMjU1IDI1NSA5NCAyNTUgMSAwIDAgMC0yNzsjMCAwIDAgMCAwCkFkZGl0aW9uYWxMaWNlbnNlRGF0YT0KTGljZW5zZVN0eWxlPW5vZGVsb2NrZWQKTGljZW5zZVN0YXJ0RGF0ZT0wNS8wOS8yMDE5CkxpY2Vuc2VEdXJhdGlvbj02ODEyCkxpY2Vuc2VFbmREYXRlPTEyLzMxLzIwMzcKTGljZW5zZUNvdW50PTEKTXVsdGlVc2VSdWxlcz0KUmVnaXN0cmF0aW9uTGV2ZWw9MwpUcnlBbmRCdXk9Tm8KU29mdFN0b3A9Tm8KQnVuZGxlPU5vCkN1c3RvbUF0dHJpYnV0ZTE9Tm8KQ3VzdG9tQXR0cmlidXRlMj1ObwpDdXN0b21BdHRyaWJ1dGUzPU5vClN1YkNhcGFjaXR5RWxpZ2libGVQcm9kdWN0PU5vClRhcmdldFR5cGU9QU5ZClRhcmdldFR5cGVOYW1lPU9wZW4gVGFyZ2V0ClRhcmdldElEPUFOWQpFeHRlbmRlZFRhcmdldFR5cGU9CkV4dGVuZGVkVGFyZ2V0SUQ9ClNlcmlhbE51bWJlcj0KVXBncmFkZT1ObwpJbnN0YWxsUHJvZ3JhbT0KQ2FwYWNpdHlUeXBlPQpNYXhPZmZsaW5lUGVyaW9kPQpEZXJpdmVkTGljZW5zZVN0eWxlPQpEZXJpdmVkTGljZW5zZVN0YXJ0RGF0ZT0KRGVyaXZlZExpY2Vuc2VFbmREYXRlPQpEZXJpdmVkTGljZW5zZUFnZ3JlZ2F0ZUR1cmF0aW9uPQo=
  account:
    privileged: true
    imagePullSecrets:
      - ibm-entitlement-key
  environment:
    database:
      name: orderdb
    dbType: db2oltp
    instance:
      password: db2inst1
  version: 11.5.6.0-cn2
  storage:
    - name: share
      type: create
      spec:
        accessModes:
          - ReadWriteMany
        resources:
          requests:
            storage: 100Gi
        storageClassName: managed-nfs-storage
EOF

# 5. Update the db2adv-vpc license
INSTANCE_NAME="db2ucluster-sample"
oc edit db2ucluster ${INSTANCE_NAME}

spec:
  license:
    value: W0xpY2Vuc2VDZXJ0aWZpY2F0ZV0KQ2hlY2tTdW09NzYwMEZGQTMyMTI1NzQyMzAxRTk2OUY1NkQ5NEZFNkUKVGltZVN0YW1wPTE1NTc0MTcxNTQKUGFzc3dvcmRWZXJzaW9uPTQKVmVuZG9yTmFtZT1JQk0gVG9yb250byBMYWIKVmVuZG9yUGFzc3dvcmQ9N3Y4cDRmcTJkdGZwYwpWZW5kb3JJRD01ZmJlZTBlZTZmZWIuMDIuMDkuMTUuMGYuNDguMDAuMDAuMDAKUHJvZHVjdE5hbWU9REIyIEFkdmFuY2VkIEVkaXRpb24KUHJvZHVjdElEPTE0MDQKUHJvZHVjdFZlcnNpb249MTEuNQpQcm9kdWN0UGFzc3dvcmQ9Ynd1NXppdXd4cXN6bmd3NWFmNHUyYjcyClByb2R1Y3RBbm5vdGF0aW9uPTEyNyAxNDMgMjU1IDI1NSA5NCAyNTUgMSAwIDAgMC0yNzsjMCAwIDAgMCAwCkFkZGl0aW9uYWxMaWNlbnNlRGF0YT0KTGljZW5zZVN0eWxlPW5vZGVsb2NrZWQKTGljZW5zZVN0YXJ0RGF0ZT0wNS8wOS8yMDE5CkxpY2Vuc2VEdXJhdGlvbj02ODEyCkxpY2Vuc2VFbmREYXRlPTEyLzMxLzIwMzcKTGljZW5zZUNvdW50PTEKTXVsdGlVc2VSdWxlcz0KUmVnaXN0cmF0aW9uTGV2ZWw9MwpUcnlBbmRCdXk9Tm8KU29mdFN0b3A9Tm8KQnVuZGxlPU5vCkN1c3RvbUF0dHJpYnV0ZTE9Tm8KQ3VzdG9tQXR0cmlidXRlMj1ObwpDdXN0b21BdHRyaWJ1dGUzPU5vClN1YkNhcGFjaXR5RWxpZ2libGVQcm9kdWN0PU5vClRhcmdldFR5cGU9QU5ZClRhcmdldFR5cGVOYW1lPU9wZW4gVGFyZ2V0ClRhcmdldElEPUFOWQpFeHRlbmRlZFRhcmdldFR5cGU9CkV4dGVuZGVkVGFyZ2V0SUQ9ClNlcmlhbE51bWJlcj0KVXBncmFkZT1ObwpJbnN0YWxsUHJvZ3JhbT0KQ2FwYWNpdHlUeXBlPQpNYXhPZmZsaW5lUGVyaW9kPQpEZXJpdmVkTGljZW5zZVN0eWxlPQpEZXJpdmVkTGljZW5zZVN0YXJ0RGF0ZT0KRGVyaXZlZExpY2Vuc2VFbmREYXRlPQpEZXJpdmVkTGljZW5zZUFnZ3JlZ2F0ZUR1cmF0aW9uPQo=

# 6. Refresh the license
oc scale --replicas=0 statefulset c-${INSTANCE_NAME}-db2u
oc scale --replicas=1 statefulset c-${INSTANCE_NAME}-db2u
oc exec -it c-db2ucluster-sample-db2u-0 -- su - db2inst1 -c "db2licm -l"

# 7. create the orderdb and populate it
oc cp ./Common/createOrderDB.sql db2/c-db2ucluster-sample-db2u-0:/tmp
oc cp ./Common/initialDataSet.sql db2/c-db2ucluster-sample-db2u-0:/tmp

oc exec -it c-db2ucluster-sample-db2u-0 -- bash
su -l db2inst1
db2 get instance
db2 db2 get connection state
db2 create database orderdb
db2 connect to orderdb
db2 -tf ./tmp/createOrderDB.sql
db2 -tf ./tmp/initialDataSet.sql

# 8. Update the liberty/server.env
DB2_HOST=172.30.203.252
#DB2_HOST=c-db2ucluster-sample-db2u-0.c-db2ucluster-sample-db2u-internal.db2.svc
DB2_PASSWORD=passw0rd

# 9. verify that db2 is listening in 50000
db2 get dbm config | grep SVCENAME
 TCP/IP Service name                          (SVCENAME) = db2c_db2inst1
grep db2c_db2inst1 /etc/services
netstat -an | grep 50000
tcp        0      0 0.0.0.0:50000           0.0.0.0:*               LISTEN

#10. get the db2 IP address and the hostname 
oc get svc -n db2
NAME                                 TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                                                                           AGE
c-db2ucluster-sample-db2u            ClusterIP   172.21.253.24    <none>        50000/TCP,50001/TCP,25000/TCP,25001/TCP,25002/TCP,25003/TCP,25004/TCP,25005/TCP   7h21m

172.21.253.24
c-db2ucluster-sample-db2u.db2.svc

###
# create the cos-liberty-tekton project
###
oc new-project cos-liberty-tekton

oc adm policy add-scc-to-user privileged -n cos-liberty-tekton -z pipeline

# import the tekton resources
cd tekton/tekton-only
oc apply -f gse-apply-manifests-pvc-task.yaml
oc apply -f gse-buildah-pvc-task.yaml
oc apply -f gse-build-deploy-pvc-pipeline.yaml
oc apply -f gse-build-pipeline-resources.yaml

# run the pipeline
tkn pipeline start  gse-build-deploy-pvc-pipeline -n cos-liberty-tekton
When prompted:
git-source http://github.com/ibm-cloud-architecture/appmod-liberty-tekton
docker-image image-registry.openshift-image-registry.svc:5000/cos-liberty-tekton/cos-liberty

