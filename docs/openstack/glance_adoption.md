# Glance adoption

Adopting Glance means that an existing `OpenStackControlPlane` CR, where Glance
is supposed to be disabled, should be patched to start the service with the
configuration parameters provided by the source environment.

When the procedure is over, the expectation is to see the `GlanceAPI` service
up and running: the `Keystone endpoints` should be updated and the same backend
of the source Cloud will be available. If the conditions above are met, the
adoption is considered concluded.

This guide also assumes that:

1. A `TripleO` environment (the source Cloud) is running on one side;
2. A `SNO` / `CodeReadyContainers` is running on the other side;
3. (optional) an internal/external `Ceph` cluster is reachable by both `crc` and
`TripleO`


## Prerequisites

* Previous Adoption steps completed. Notably, MariaDB and Keystone
  should be already adopted.

## Procedure - Glance adoption

As already done for [Keystone](https://github.com/openstack-k8s-operators/data-plane-adoption/blob/main/keystone_adoption.md), the Glance Adoption follows the same pattern.

### Using local storage backend

When Glance should be deployed with local storage backend (not Ceph),
patch OpenStackControlPlane to deploy Glance:

```
oc patch openstackcontrolplane openstack --type=merge --patch '
spec:
  glance:
    enabled: true
    apiOverride:
      route: {}
    template:
      databaseInstance: openstack
      storageClass: "local-storage"
      storageRequest: 10G
      glanceAPIInternal:
        override:
          service:
            metadata:
              annotations:
                metallb.universe.tf/address-pool: internalapi
                metallb.universe.tf/allow-shared-ip: internalapi
                metallb.universe.tf/loadBalancerIPs: 172.17.0.80
            spec:
              type: LoadBalancer
        networkAttachments:
        - storage
      glanceAPIExternal:
        networkAttachments:
        - storage
'
```

### Using Ceph storage backend

If a Ceph backend is used, the `customServiceConfig` parameter should
be used to inject the right configuration to the `GlanceAPI` instance.

Make sure the Ceph-related secret (`ceph-conf-files`) was created in
the `openstack` namespace and that the `extraMounts` property of the
`OpenStackControlPlane` CR has been configured properly. These tasks
are described in an earlier Adoption step [Ceph storage backend
configuration](ceph_backend_configuration.md).

```bash
cat << EOF > glance_patch.yaml
spec:
  glance:
    enabled: true
    template:
      databaseInstance: openstack
      customServiceConfig: |
        [DEFAULT]
        enabled_backends=default_backend:rbd
        [glance_store]
        default_backend=default_backend
        [default_backend]
        rbd_store_ceph_conf=/etc/ceph/ceph.conf
        rbd_store_user=openstack
        rbd_store_pool=images
        store_description=Ceph glance store backend.
      storageClass: "local-storage"
      storageRequest: 10G
      glanceAPIInternal:
        override:
          service:
            metadata:
              annotations:
                metallb.universe.tf/address-pool: internalapi
                metallb.universe.tf/allow-shared-ip: internalapi
                metallb.universe.tf/loadBalancerIPs: 172.17.0.80
            spec:
              type: LoadBalancer
        networkAttachments:
        - storage
      glanceAPIExternal:
        networkAttachments:
        - storage
EOF
```

> If you have previously backup your Openstack services configuration file from the old environment:
[pull openstack configuration os-diff](pull_openstack_configuration.md) you can use os-diff to compare
and make sure the configuration is correct.

```bash
pushd os-diff
./os-diff cdiff --service glance -c /tmp/collect_tripleo_configs/glance/etc/glance/glance-api.conf -o glance_patch.yaml
```

> This will producre the difference between both ini configuration files.

Patch OpenStackControlPlane to deploy Glance with Ceph backend:

```
oc patch openstackcontrolplane openstack --type=merge --patch-file glance_patch.yaml
```

## Post-checks

### Test the glance service from the OpenStack CLI

> You can compare and make sure the configuration has been correctly applied to the glance pods by running

```bash
./os-diff cdiff --service glance -c /etc/glance/glance.conf.d/02-config.conf  -o glance_patch.yaml --frompod -p glance-api
```

> If no line appear, then the configuration is correctly done.

Inspect the resulting glance pods:

```bash
GLANCE_POD=`oc get pod |grep glance-external-api | cut -f 1 -d' '`
oc exec -t $GLANCE_POD -c glance-api -- cat /etc/glance/glance.conf.d/02-config.conf

[DEFAULT]
enabled_backends=default_backend:rbd
[glance_store]
default_backend=default_backend
[default_backend]
rbd_store_ceph_conf=/etc/ceph/ceph.conf
rbd_store_user=openstack
rbd_store_pool=images
store_description=Ceph glance store backend.

oc exec -t $GLANCE_POD -c glance-api -- ls /etc/ceph
ceph.client.openstack.keyring
ceph.conf
```

Ceph secrets are properly mounted, at this point let's move to the OpenStack
CLI and check the service is active and the endpoints are properly updated.


```
(openstack)$ service list | grep image

| fc52dbffef36434d906eeb99adfc6186 | glance    | image        |

(openstack)$ endpoint list | grep image

| 569ed81064f84d4a91e0d2d807e4c1f1 | regionOne | glance       | image        | True    | internal  | http://glance-internal-openstack.apps-crc.testing   |
| 5843fae70cba4e73b29d4aff3e8b616c | regionOne | glance       | image        | True    | public    | http://glance-public-openstack.apps-crc.testing     |
| 709859219bc24ab9ac548eab74ad4dd5 | regionOne | glance       | image        | True    | admin     | http://glance-admin-openstack.apps-crc.testing      |
```

Check the images that we previously listed in the source Cloud are available
in the adopted service:

```
(openstack)$ image list
+--------------------------------------+--------+--------+
| ID                                   | Name   | Status |
+--------------------------------------+--------+--------+
| c3158cad-d50b-452f-bec1-f250562f5c1f | cirros | active |
+--------------------------------------+--------+--------+
```

### Image upload

We can test that an image can be created on from the adopted service.

```
(openstack)$ alias openstack="oc exec -t openstackclient -- openstack"
(openstack)$ curl -L -o /tmp/cirros-0.5.2-x86_64-disk.img http://download.cirros-cloud.net/0.5.2/cirros-0.5.2-x86_64-disk.img
    qemu-img convert -O raw /tmp/cirros-0.5.2-x86_64-disk.img /tmp/cirros-0.5.2-x86_64-disk.img.raw
    openstack image create --container-format bare --disk-format raw --file /tmp/cirros-0.5.2-x86_64-disk.img.raw cirros2
    openstack image list
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   273  100   273    0     0   1525      0 --:--:-- --:--:-- --:--:--  1533
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0
100 15.5M  100 15.5M    0     0  17.4M      0 --:--:-- --:--:-- --:--:-- 17.4M

+------------------+--------------------------------------------------------------------------------------------------------------------------------------------+
| Field            | Value                                                                                                                                      |
+------------------+--------------------------------------------------------------------------------------------------------------------------------------------+
| container_format | bare                                                                                                                                       |
| created_at       | 2023-01-31T21:12:56Z                                                                                                                       |
| disk_format      | raw                                                                                                                                        |
| file             | /v2/images/46a3eac1-7224-40bc-9083-f2f0cd122ba4/file                                                                                       |
| id               | 46a3eac1-7224-40bc-9083-f2f0cd122ba4                                                                                                       |
| min_disk         | 0                                                                                                                                          |
| min_ram          | 0                                                                                                                                          |
| name             | cirros                                                                                                                                     |
| owner            | 9f7e8fdc50f34b658cfaee9c48e5e12d                                                                                                           |
| properties       | os_hidden='False', owner_specified.openstack.md5='', owner_specified.openstack.object='images/cirros', owner_specified.openstack.sha256='' |
| protected        | False                                                                                                                                      |
| schema           | /v2/schemas/image                                                                                                                          |
| status           | queued                                                                                                                                     |
| tags             |                                                                                                                                            |
| updated_at       | 2023-01-31T21:12:56Z                                                                                                                       |
| visibility       | shared                                                                                                                                     |
+------------------+--------------------------------------------------------------------------------------------------------------------------------------------+

+--------------------------------------+--------+--------+
| ID                                   | Name   | Status |
+--------------------------------------+--------+--------+
| 46a3eac1-7224-40bc-9083-f2f0cd122ba4 | cirros2| active |
| c3158cad-d50b-452f-bec1-f250562f5c1f | cirros | active |
+--------------------------------------+--------+--------+


(openstack)$ oc rsh ceph
sh-4.4$ ceph -s
r  cluster:
    id:     432d9a34-9cee-4109-b705-0c59e8973983
    health: HEALTH_OK

  services:
    mon: 1 daemons, quorum a (age 4h)
    mgr: a(active, since 4h)
    osd: 1 osds: 1 up (since 4h), 1 in (since 4h)

  data:
    pools:   5 pools, 160 pgs
    objects: 46 objects, 224 MiB
    usage:   247 MiB used, 6.8 GiB / 7.0 GiB avail
    pgs:     160 active+clean

sh-4.4$ rbd -p images ls
46a3eac1-7224-40bc-9083-f2f0cd122ba4
c3158cad-d50b-452f-bec1-f250562f5c1f
```
