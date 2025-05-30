# Welcome to base chart for spring boot applications

## This is a common chart which can be used as base for any spring applications
### It can be added as classic repository to you Chart file with command 
```shell
helm repo add $reponame https://raw.githubusercontent.com/Merdoc97/spring-boot-helm/master
```
### It include out of the box:
#### retry template for ingress
- by default it retry for 502,503,504 codes and has 3 retries
- here is example how it can be overridden
```yaml
global:
  application:
    ingress:
      annotations:
        nginx.ingress.kubernetes.io/proxy-next-upstream-tries: "5"
```
#### migration hook: 
##### you can turn migration hook as shown in example bellow
```yaml

```
