# Demo Pod Presets

## Configuring Openshift environment (on Minishift)
**Note** Pod Presets plugin needs to be added
Add the plug-in using the following configuration:

````
admissionConfig:
  pluginConfig:
    PodPreset:
      configuration:
        kind: DefaultAdmissionConfig
        apiVersion: v1
        disable: false
````

 'minishift ssh' and editing the master config via 'vi /var/lib/minishift/openshift.local.config/master/master-config.yaml'



## Configure Minishift

````
export MINISHIFT_ENABLE_EXPERIMENTAL=true

minishift start --iso-url=https://github.com/minishift/minishift-centos-iso/releases/download/v1.3.0/minishift-centos7.iso --vm-driver=virtualbox   --cpus=4 --memory=8192 --service-catalog

oc login -u system:admin

oc adm policy add-cluster-role-to-group system:openshift:templateservicebroker-client system:unauthenticated system:authenticated

````




## Create the pod preset resource definition 

```
cat env-presets.yaml.yaml
kind: PodPreset
apiVersion: settings.k8s.io/v1alpha1
metadata:
  name: environment-injection-dev
spec:
  selector:
    matchLabels:
      apptype: springapp
  envFrom:
    - configMapRef:
        name: envconfig
```

```
oc create -f env-presets.yaml
```
### Create the configmap

```
oc create configmap envconfig --from-literal=SPRING_PROFILES_ACTIVE=dev
```


## Create the application 
```
oc import-image openshift/openjdk18-openshift --from=registry.access.redhat.com/redhat-openjdk-18/openjdk18-openshift --confirm

oc new-build -i openjdk18-openshift --binary=true --name='noctest'

mnv clean package -DskipTests

oc start-build noctest --from-file=./target/booster-rest-http-spring-boot-1.0.0-SNAPSHOT-exec.jar
```

## Add labels to the application which drive the pod preset injection
```
oc new-app noctest -l apptype=springapp
```

## Testing

In the SpringBoot logs the profile selection is output as follows

```
  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::        (v1.4.1.RELEASE)
2017-11-25 20:19:38.520  INFO 1 --- [           main] io.openshift.booster.BoosterApplication  : The following profiles are active: production
```


With a different environment profile specified e.g. test
```
oc create configmap envconfig --from-literal=SPRING_PROFILES_ACTIVE=test
``` 

The test profile is different
```
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
@@@@THIS IS THE TEST ENVIRONMENT@@@@
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
2017-11-26 03:14:01.455  INFO 1 --- [           main] io.openshift.booster.BoosterApplication  : The following profiles are active: test
```

Likewise for the dev profile

```
oc create configmap envconfig --from-literal=SPRING_PROFILES_ACTIVE=dev
```

```
.........................................
...THIS IS THE DEVELOPMENT ENVIRONMENT...
.........................................
2017-11-26 03:28:15.175  INFO 1 --- [           main] io.openshift.booster.BoosterApplication  : The following profiles are active: dev
```


