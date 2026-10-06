# cloud-pak-deployer-storage-fusion

This repo is using [Cloud pak deployer](https://ibm.github.io/cloud-pak-deployer).

This repo has two purposes:
1. Build the Cloud pak deployed in Openshift making it available to deploying Cloud paks.
2. Containing the configuration that will be used by the Cloud pak deployer

## Instructions

1. Fork this repo if you want to change the configuration in /config. Make sure to update the [URL to the git repo](https://github.com/thomas-mattsson/cloud-pak-deployer-storage-fusion/blob/main/resources/resources.yaml#L7) if doing so.
2. If forked, update the configuration files in /config to match the wanted environment. See details in the cloud pak deployer instructions.
3. Run the following commands to build cloud pak deployer
```bash
oc create -f https://raw.githubusercontent.com/bganslandt/cloud-pak-deployer-storage-fusion/orchestrate-540/resources/build.yaml
```
4. Add your entitlement key in a secret using the following command. You will need to insert your IBM entitlement registry key where indicated.
```bash
oc create secret generic cloud-pak-deployer-input -n cloud-pak-deployer --from-literal entitlement-key="<your entitlement key>"
```
5. Wait until the build from step 3 is finished. You can track it in the Builds/Builds section in the Openshift console.
6. Run the following command to start the deployment job using the configuration from the `config` directory.
```bash
oc create -f https://raw.githubusercontent.com/bganslandt/cloud-pak-deployer-storage-fusion/orchestrate-540/resources/resources.yaml
```

Logs can be followed with
```bash
oc logs -f -n cloud-pak-deployer job/cloud-pak-deployer

# Create cpd debug pod
cat << EOF | oc apply -f -
apiVersion: batch/v1
kind: Job
metadata:
  labels:
    app: cloud-pak-deployer-debug
  name: cloud-pak-deployer-debug
  namespace: cloud-pak-deployer
spec:
  parallelism: 1
  completions: 1
  backoffLimit: 0
  template:
    metadata:
      name: cloud-pak-deployer-debug
      labels:
        app: cloud-pak-deployer-debug
    spec:
      containers:
      - name: cloud-pak-deployer-debug
        image: quay.io/cloud-pak-deployer/cloud-pak-deployer:latest
        imagePullPolicy: Always
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
        env:
        - name: CONFIG_DIR
          value: /Data/cpd-config
        - name: STATUS_DIR
          value: /Data/cpd-status
        volumeMounts:
        - name: config-volume
          mountPath: /Data/cpd-config/config
        - name: status-volume
          mountPath: /Data/cpd-status
        command: ["/bin/sh","-xc"]
        args: 
          - sleep infinity
      restartPolicy: Never
      securityContext:
        runAsUser: 0
      serviceAccountName: cloud-pak-deployer-sa
      volumes:
      - name: config-volume
        configMap:
          name: cloud-pak-deployer-input
      - name: status-volume
        persistentVolumeClaim:
          claimName: cloud-pak-deployer-status        
EOF

oc -n cloud-pak-deployer exec $(oc get po -n cloud-pak-deployer -oname | findstr "debug") -it -- /bin/bash
cd /Data/cpd-status/log/ && echo '===> LOGS: ===' && ls -lArt && echo '==============' && tail -n 50 -f $(ls -Art | grep -v 'cloud-pak-deployer.log' | tail -1)

```
