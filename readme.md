# Pod Presets - Dynamic Configuration Demo
The purpose of this demo is to show how _k8s pod presets_ can be used to drive different configuration settings in an application depending on the environment.

In this demo the SpringBoot application has different runtime configurations depending on what SpringBoot profile is active. 

The profile is selected by setting the Spring environment variable _SPRING_PROFILES_ACTIVE_

With this approach it's possible to add configuration/functionality for multiple environments in the application image and select which one is active via environment variables at runtime.

### Benefits  
* Configmaps are mounted based on labels & selectors rather than being explicitly mounted via commands
* A single config map per environment set by the administrator drives the settings of profiles
* Additional config maps can be added as needed

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

 To edit the configuration use: 
1. 'minishift ssh' and 
1. Edit the master config via 'vi /var/lib/minishift/openshift.local.config/master/master-config.yaml'



## Start Minishift

````
$export MINISHIFT_ENABLE_EXPERIMENTAL=true

$minishift start --iso-url=https://github.com/minishift/minishift-centos-iso/releases/download/v1.3.0/minishift-centos7.iso --vm-driver=virtualbox   --cpus=4 --memory=8192 --service-catalog

$oc login -u system:admin

$oc adm policy add-cluster-role-to-group system:openshift:templateservicebroker-client system:unauthenticated system:authenticated

````

## Create the pod preset resource definition 

The key here is the "matchLabels" section as this is key to how the preset is applied to the pods.

This _podpreset_ will create environment variables in the relevant pods from the contents of the associated configmap. In our example the environment variable _SPRING_PROFILES_ACTIVE_ is created.

```
$cat env-presets.yaml

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
$oc create -f env-presets.yaml
```
### Create the configmap

```
$oc create configmap envconfig --from-literal=SPRING_PROFILES_ACTIVE=dev
```


## Build the application and create the application 
```
$oc import-image openshift/openjdk18-openshift --from=registry.access.redhat.com/redhat-openjdk-18/openjdk18-openshift --confirm

$oc new-build -i openjdk18-openshift --binary=true --name='presettest'

$mnv clean package -DskipTests

$oc start-build presettest --from-file=./target/booster-rest-http-spring-boot-1.0.0-SNAPSHOT-exec.jar
```

## Add labels to the application which drive the pod preset injection

Note how this maps to the "matchLabels" section of the pod presets.
```
$oc new-app presettest -l apptype=springapp
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
$oc create configmap envconfig --from-literal=SPRING_PROFILES_ACTIVE=test
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
$oc create configmap envconfig --from-literal=SPRING_PROFILES_ACTIVE=dev
```

```
.........................................
...THIS IS THE DEVELOPMENT ENVIRONMENT...
.........................................
2017-11-26 03:28:15.175  INFO 1 --- [           main] io.openshift.booster.BoosterApplication  : The following profiles are active: dev
```


