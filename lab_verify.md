Verify 

```
oc get agentserviceconfig agent -o jsonpath={.spec.osImages} | jq
```

> [<br>
>  nbsp; nbsp;{<br>
>  nbsp; nbsp; nbsp; nbsp;"cpuArchitecture": nbsp;"x86_64",<br>
>  nbsp; nbsp; nbsp; nbsp;"openshiftVersion": nbsp;"4.14",<br>
>  nbsp; nbsp; nbsp; nbsp;"rootFSUrl": nbsp;"http://infra.5g-deployment.lab:8080/rhcos-4.14.0-x86_64-live-rootfs.x86_64.img",<br>
>  nbsp; nbsp; nbsp; nbsp;"url": nbsp;"http://infra.5g-deployment.lab:8080/rhcos-4.14.0-x86_64-live.x86_64.iso",<br>
>  nbsp; nbsp; nbsp; nbsp;"version": nbsp;"414.92.202310210434-0"<br>
>  nbsp; nbsp;}<br>
> ]<br>
