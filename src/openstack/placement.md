release time :2019-04-18 09:14

The main content of this article is transferred from the article of Huawei Fan Guixiong.  

# background
Private cloud users, especially private cloud users undergoing traditional IT architecture transformation, generally have a variety of stock resource systems, and connecting with these systems will complicate the OpenStack resource system.

From the user's point of view, you may want to:

As a user of shared storage solutions, you would expect Nova and Horizon to correctly report the total and usage information of shared storage disk resources.
As an advanced Neutron user, it is expected to use an external third-party routing network function. I hope that Nova can grasp and use a specific network port to associate with a specific subnet pool to ensure that the virtual machine can be started on the subnet pool.
As an advanced Cinder user, I hope that when I specify cinder volume-id in the nova boot command, Nova can know which computing nodes are associated with the Cinder storage pool where the Request Volume is located.
Therefore, in addition to processing internal resources such as computing node CPU, memory, PCI devices, and local disks, OpenStack often needs to manage external resources such as storage services provided by SDS and NFS, and network services provided by SDN.

But in the past, Nova can only handle resources provided by computing nodes. Nova Resource Tracker assumes that all resources come from computing nodes. Therefore, when reporting resource status periodically, Resource Tracker will simply add statistics on the total resources and usage of the computing node list. Obviously, this cannot meet the complex production requirements mentioned above, and also violates the openness principle that OpenStack has always been proud of. And as the definition of OpenStack is further upgraded by the community to "an open source infrastructure integration engine", it means that the resource system of OpenStack will be composed of more external resource types.

Therefore, when resource types and providers become diverse, a highly abstract and simple and unified management method is naturally required to allow users and codes to easily use, manage, and monitor the entire OpenStack system resources. This is Placement (layout) ).

# Project Description


Placement shoulders such a historical mission. It was first introduced into the openstack/nova repo in the Newton version and incubated in the form of API, so it is often called Placement API. It participates in the scheduling process of nova-scheduler selecting the target host, is responsible for tracking and recording the Inventory and Usage of the Resource Provider, and uses different Resource Classes to classify resource types, and uses different Resource Traits to mark resource characteristics.

The Ocata version of the Placement API is optional, and users are advised to enable and replace CpuFilter, CoreFilter, and DiskFilter. The Pike version is mandatory to start the Placement API service, otherwise the nova-compute service cannot run normally.

Placement API started the openstack/nova repo spin-off process, transformed from Placement API to OpenStack Placement, and became a separate project in the Stein release.

# community development
Starting from the S version, Placement has released the first official version 1.0.0. The Placement code is hosted in its own warehouse and managed as an independent OpenStack project.

Most of the changes in the S version are not user-oriented, but the migration of internal data from nova to placement.

The S version has a total of 338 commits, the following is the company's participation.

https://www.stackalytics.com/?module=placement&release=stein&metric=commits

    Contribution Summary
    Commits: 338
    LOCs: 61838
    Do not merge (-2): 2
    Patch needs further work (-1): 154
    Looks good (+1): 313
    Looks good for core (+2): 649
    Approve: 305
    Abandon: 0
    Change Requests: 350 (30 of them abandoned)
    Patch Sets: 1320
    Draft Blueprints: 0
    Completed Blueprints: 0
    Filed Bugs: 0
    Resolved Bugs: 0
    Emails: 0
    Translations: 0


#working principle
nova-compute reports resource information to placement. Nova-scheduler still uses partially reserved filters and weights while asking placement for nodes that satisfy a series of resource requests. (Currently placement only replaces the commonly used filters of nova-scheduler's CpuFilter, CoreFilter and DiskFilter)

Nova-scheduler makes two calls to placement-api. For the first time, nova-scheduler obtains a set of Allocation Candidates (allocation candidates) from placement-api. The so-called Allocation Candidates are Resource Providers that can meet resource requirements.

EXAMPLE:

    GET /allocation_candidates?resources=VCPU:1,MEMORY_MB:2048,DISK_GB:100


NOTE: The implementation of obtaining Allocation Candidates is a series of complex database cascading query and filtering operations, using query params as filtering conditions. This example passes the vCPU, RAM, and Disk resources required by the Launch Instance. In addition, the required and member_of parameters can also be provided to specify the Resource Traits and Resource Provider Aggregate characteristics, making the acquisition of Allocation Candidates more flexible. See Allocation candidates for more details.

    [root@control01 ~]# openstack allocation candidate list --resource VCPU=1,MEMORY_MB=2048,DISK_GB=10 --required HW_CPU_X86_SSE2
    +---+----------------------------------+--------------------------------------+----------------------------------------------+--------------------------------------------------------------+
    | # | allocation                       | resource provider                    | inventory used/capacity                      | traits                                                       |
    +---+----------------------------------+--------------------------------------+----------------------------------------------+--------------------------------------------------------------+
    | 1 | VCPU=1,MEMORY_MB=2048,DISK_GB=10 | 5c5a578f-51b0-481c-b38c-7aaa3394e585 | VCPU=5/512,MEMORY_MB=3648/60670,DISK_GB=7/49 | HW_CPU_X86_SSE2,HW_CPU_X86_SSE,HW_CPU_X86_MMX,HW_CPU_X86_SVM |
    +---+----------------------------------+--------------------------------------+----------------------------------------------+--------------------------------------------------------------+

The JSON object with a list of allocation requests and a JSON object of provider summary objects returned by placement-api to nova-scheduler has the following data structure, the key lies in the two fields of allocation_requests and provider_summaries, which also play an important role in the subsequent Scheduler Filters logic role.

    {
    "allocation_requests": [
        <ALLOCATION_REQUEST_1>,
        ...
        <ALLOCATION_REQUEST_N>
    ],
    "provider_summaries": {
        <COMPUTE_NODE_UUID_1>: <PROVIDER_SUMMARY_1>,
        ...
        <COMPUTE_NODE_UUID_N>: <PROVIDER_SUMMARY_N>,
    }
    }


allocation_requests: Contains a list of all resource providers that can meet the requirements and their expected allocation resources.

    "allocation_requests": [
            {
                "allocations": {
                    "a99bad54-a275-4c4f-a8a3-ac00d57e5c64": {
                        "resources": {
                            "DISK_GB": 100
                        }
                    },
                    "35791f28-fb45-4717-9ea9-435b3ef7c3b3": {
                        "resources": {
                            "VCPU": 1,
                            "MEMORY_MB": 1024
                        }
                    }
                }
            },
            {
                "allocations": {
                    "a99bad54-a275-4c4f-a8a3-ac00d57e5c64": {
                        "resources": {
                            "DISK_GB": 100
                        }
                    },
                    "915ef8ed-9b91-4e38-8802-2e4224ad54cd": {
                        "resources": {
                            "VCPU": 1,
                            "MEMORY_MB": 1024
                        }
                    }
                }
            }
        ],

provider_summaries: Contains the total resources and usage information of all resource providers that meet the requirements.

    "provider_summaries": {
            "a99bad54-a275-4c4f-a8a3-ac00d57e5c64": {
                "resources": {
                    "DISK_GB": {
                        "used": 0,
                        "capacity": 1900
                    }
                },
                "traits": ["MISC_SHARES_VIA_AGGREGATE"],
                "parent_provider_uuid": null,
                "root_provider_uuid": "a99bad54-a275-4c4f-a8a3-ac00d57e5c64"
            },
            "35791f28-fb45-4717-9ea9-435b3ef7c3b3": {
                "resources": {
                    "VCPU": {
                        "used": 0,
                        "capacity": 384
                    },
                    "MEMORY_MB": {
                        "used": 0,
                        "capacity": 196608
                    }
                },
                "traits": ["HW_CPU_X86_SSE2", "HW_CPU_X86_AVX2"],
                "parent_provider_uuid": null,
                "root_provider_uuid": "35791f28-fb45-4717-9ea9-435b3ef7c3b3"
            },
            "915ef8ed-9b91-4e38-8802-2e4224ad54cd": {
                "resources": {
                    "VCPU": {
                        "used": 0,
                        "capacity": 384
                    },
                    "MEMORY_MB": {
                        "used": 0,
                        "capacity": 196608
                    }
                },
                "traits": ["HW_NIC_SRIOV"],
                "parent_provider_uuid": null,
                "root_provider_uuid": "915ef8ed-9b91-4e38-8802-2e4224ad54cd"
            },
            "f5120cad-67d9-4f20-9210-3092a79a28cf": {
                "resources": {
                    "SRIOV_NET_VF": {
                        "used": 0,
                        "capacity": 8
                    }
                },
                "traits": [],
                "parent_provider_uuid": "915ef8ed-9b91-4e38-8802-2e4224ad54cd",
                "root_provider_uuid": "915ef8ed-9b91-4e38-8802-2e4224ad54cd"
            }
        }


NOTE: It can be seen that SRIOV_NET_VF is also regarded as a resource type, provided by a dedicated resource provider.



After nova-scheduler obtains the Allocation Candidates, it further uses the Filtered and Weighed mechanisms to finally determine the target host. Then deduct (claim_resources) the resource usage of the resource provider corresponding to the target host according to the data of allocation requests and provider summaries, which is what nova-scheduler calls placement-api for the second time. Review the contents of the allocations tables:

    MariaDB [nova_api]> select * from allocations;
    +---------------------+------------+----+----------------------+--------------------------------------+-------------------+------+
    | created_at          | updated_at | id | resource_provider_id | consumer_id                          | resource_class_id | used |
    +---------------------+------------+----+----------------------+--------------------------------------+-------------------+------+
    | 2018-08-01 10:52:15 | NULL       |  7 |                    1 | f8d55035-389c-47b8-beea-02f00f25f5d9 |                 0 |    1 |
    | 2018-08-01 10:52:15 | NULL       |  8 |                    1 | f8d55035-389c-47b8-beea-02f00f25f5d9 |                 1 |  512 |
    | 2018-08-01 10:52:15 | NULL       |  9 |                    1 | f8d55035-389c-47b8-beea-02f00f25f5d9 |                 2 |    1 |
    +---------------------+------------+----+----------------------+--------------------------------------+-------------------+------+
    # consumer_id consumer
    # resource_class_id resource type
    # resource_provider_id resource provider
    # used allocated amount
    # The above records indicate that 1 vCPU, RAM 512MB, and Disk 1GB are allocated to the virtual machine
    Obviously, the Consumer consumer is the virtual machine to be created.

# Software Installation
1 Create a database

To create a database, complete the following steps:

**Connect to the database server as root using the database access client:**

    $ mysql -u root -p


**Grant appropriate access to the database:**

    MariaDB [(none)]> CREATE DATABASE placement;

**Exit the database access client.**

2 Configure users and endpoints

1)Source admin credentials to access admin-only CLI commands:

    $ .admin-openrc

2)Create the Placement service user PLACEMENT_PASS with your choice:

$ openstack user create --domain default --password-prompt placement

    User Password:
    Repeat User Password:
    +---------------------+----------------------------------+
    | Field               | Value                            |
    +---------------------+----------------------------------+
    | domain_id           | default                          |
    | enabled             | True                             |
    | id                  | fa742015a6494a949f67629884fc7ec8 |
    | name                | placement                        |
    | options             | {}                               |
    | password_expires_at | None                             |
    +---------------------+----------------------------------+


3)Add the Placement user to the service project with the admin role:

    $ openstack role add –project service –user placement admin

4)Create a Placement API entry in the service catalog:

    $ openstack service create --name placement \
    --description "Placement API" placement

    +-------------+----------------------------------+
    | Field       | Value                            |
    +-------------+----------------------------------+
    | description | Placement API                    |
    | enabled     | True                             |
    | id          | 2d1a27022e6e4185b86adac4444c495f |
    | name        | placement                        |
    | type        | placement                        |
    +-------------+----------------------------------+

5)Create the Placement API service endpoint:

    $ openstack endpoint create --region RegionOne \
    placement public http://controller:8778

    +--------------+----------------------------------+
    | Field        | Value                            |
    +--------------+----------------------------------+
    | enabled      | True                             |
    | id           | 2b1b2637908b4137a9c2e0470487cbc0 |
    | interface    | public                           |
    | region       | RegionOne                        |
    | region_id    | RegionOne                        |
    | service_id   | 2d1a27022e6e4185b86adac4444c495f |
    | service_name | placement                        |
    | service_type | placement                        |
    | url          | http://controller:8778           |
    +--------------+----------------------------------+

    $ openstack endpoint create --region RegionOne \
    placement internal http://controller:8778

    +--------------+----------------------------------+
    | Field        | Value                            |
    +--------------+----------------------------------+
    | enabled      | True                             |
    | id           | 02bcda9a150a4bd7993ff4879df971ab |
    | interface    | internal                         |
    | region       | RegionOne                        |
    | region_id    | RegionOne                        |
    | service_id   | 2d1a27022e6e4185b86adac4444c495f |
    | service_name | placement                        |
    | service_type | placement                        |
    | url          | http://controller:8778           |
    +--------------+----------------------------------+

    $ openstack endpoint create --region RegionOne \
    placement admin http://controller:8778

    +--------------+----------------------------------+
    | Field        | Value                            |
    +--------------+----------------------------------+
    | enabled      | True                             |
    | id           | 3d71177b9e0f406f98cbff198d74b182 |
    | interface    | admin                            |
    | region       | RegionOne                        |
    | region_id    | RegionOne                        |
    | service_id   | 2d1a27022e6e4185b86adac4444c495f |
    | service_name | placement                        |
    | service_type | placement                        |
    | url          | http://controller:8778           |
    +--------------+----------------------------------+


3 Install and configure components

1)Installation package:

    yum install openstack-placement-api

Edit the /etc/placement/placement.conf file and complete the following:

2)In the [placement_database] section, configure database access:

    [placement_database]

    …
    connection = mysql+pymysql://placement:PLACEMENT_DBPASS@controller/placement
    replace PLACEMENT_DBPASS with the password you chose for the placement database.

3)In the [api] and [keystone_authtoken] sections, configure the identity service access:

    [api]
    # ...
    auth_strategy = keystone

    [keystone_authtoken]
    # ...
    auth_url = http://controller:5000/v3
    memcached_servers = controller:11211
    auth_type = password
    project_domain_name = default
    user_domain_name = default
    project_name = service
    username = placement
    password = PLACEMENT_PASS

4 Populate the placement database:

    su -s /bin/sh -c "placement-manage db sync" placement

5 Complete the installation

Restart the httpd service


    systemctl restart httpd


Instructions
1 Resource provider aggregates function

Resource provider aggregates is a function similar to Host Aggregate. When obtaining Allocation Candidates, it supports obtaining from a specific Aggregate through the member_of request query parameter. Resource provider aggregates are very suitable for production scenarios with different types of host aggregation (eg high-performance host aggregation, large storage capacity host aggregation).

Create resource provider aggregates

    [root@control01 ~]# openstack aggregate create --zone nova host_aggregate_1
    +-------------------+----------------------------+
    | Field             | Value                      |
    +-------------------+----------------------------+
    | availability_zone | nova                       |
    | created_at        | 2018-12-08T05:49:55.051678 |
    | deleted           | False                      |
    | deleted_at        | None                       |
    | id                | 1                          |
    | name              | host_aggregate_1           |
    | updated_at        | None                       |
    +-------------------+----------------------------+

    [root@control01 ~]# openstack aggregate add host host_aggregate_1 control01
    +-------------------+---------------------------------+
    | Field             | Value                           |
    +-------------------+---------------------------------+
    | availability_zone | nova                            |
    | created_at        | 2018-12-08T05:49:55.000000      |
    | deleted           | False                           |
    | deleted_at        | None                            |
    | hosts             | [u'control01']                  |
    | id                | 1                               |
    | metadata          | {u'availability_zone': u'nova'} |
    | name              | host_aggregate_1                |
    | updated_at        | None                            |
    +-------------------+---------------------------------+

    [root@control01 ~]# openstack aggregate show host_aggregate_1
    +-------------------+----------------------------+
    | Field             | Value                      |
    +-------------------+----------------------------+
    | availability_zone | nova                       |
    | created_at        | 2018-12-08T05:49:55.000000 |
    | deleted           | False                      |
    | deleted_at        | None                       |
    | hosts             | [u'control01']             |
    | id                | 1                          |
    | name              | host_aggregate_1           |
    | properties        |                            |
    | updated_at        | None                       |
    +-------------------+----------------------------+

    [root@control01 ~]# openstack resource provider list
    +--------------------------------------+-----------+------------+--------------------------------------+----------------------+
    | uuid                                 | name      | generation | root_provider_uuid                   | parent_provider_uuid |
    +--------------------------------------+-----------+------------+--------------------------------------+----------------------+
    | 5c5a578f-51b0-481c-b38c-7aaa3394e585 | control01 |         26 | 5c5a578f-51b0-481c-b38c-7aaa3394e585 | None                 |
    +--------------------------------------+-----------+------------+--------------------------------------+----------------------+

    [root@control01 ~]# openstack resource provider aggregate list 5c5a578f-51b0-481c-b38c-7aaa3394e585
    +--------------------------------------+
    | uuid                                 |
    +--------------------------------------+
    | 5eea7084-0207-44f0-bbeb-c759e8c766a1 |
    +--------------------------------------+

List allocation candidates filter by aggregates

    # REQ
    curl -i "http://172.18.22.222/placement/allocation_candidates?resources=VCPU:1,MEMORY_MB:512,DISK_GB:5&member_of=5eea7084-0207-44f0-bbeb-c759e8c766a1" \
    -X GET \
    -H 'Content-type: application/json' \
    -H 'Accept: application/json' \
    -H 'X-Auth-Project-Id: admin' \
    -H 'OpenStack-API-Version: placement 1.21' \
    -H 'X-Auth-Token:gAAAAABcC12qN3GdLvjYXSSUODi7Dg9jTHUfcnF7I_ljmcffZjs3ignipGLj6iqDvDJ1gXkzGIDW6rRRNcXary-wPfgsb3nCWRIEiAS8LrReI4SYL1KfQiGW7j92b6zTz7RoSEBXACQ9z7UUVfeJ06n8WqVMBaSob4BeFIuHiVKpYCJNv7LR6cI'

    # RESP
    {
        "provider_summaries": {
            "5c5a578f-51b0-481c-b38c-7aaa3394e585": {
                "traits": ["HW_CPU_X86_SSE2", "HW_CPU_X86_SSE", "HW_CPU_X86_MMX", "HW_CPU_X86_SVM"],
                "resources": {
                    "VCPU": {
                        "used": 5,
                        "capacity": 512
                    },
                    "MEMORY_MB": {
                        "used": 3648,
                        "capacity": 60670
                    },
                    "DISK_GB": {
                        "used": 7,
                        "capacity": 49
                    }
                }
            }
        },
        "allocation_requests": [{
            "allocations": {
                "5c5a578f-51b0-481c-b38c-7aaa3394e585": {
                    "resources": {
                        "VCPU": 1,
                        "MEMORY_MB": 512,
                        "DISK_GB": 5
                    }
                }
            }
        }]
    }


2 Resource traits function

Resource traits feature label function, used to identify the feature properties of Resource Provider, each Resource Provider has its own default traits, and also supports custom traits for the specified Resource Provider.

Resource traits is a very flexible design, similar to the role of "tags". Users can build a "tag cloud" and decide to put a "tag" on a Resource Provider. It is an auxiliary tool for resource classification requirements.

    List traits
    curl -i "http://172.18.22.222/placement/traits" \
    -X GET \
    -H 'Content-type: application/json' \
    -H 'Accept: application/json' \
    -H 'X-Auth-Project-Id: admin' \
    -H 'OpenStack-API-Version: placement 1.21' \
    -H 'X-Auth-Token:gAAAAABcC12qN3GdLvjYXSSUODi7Dg9jTHUfcnF7I_ljmcffZjs3ignipGLj6iqDvDJ1gXkzGIDW6rRRNcXary-wPfgsb3nCWRIEiAS8LrReI4SYL1KfQiGW7j92b6zTz7RoSEBXACQ9z7UUVfeJ06n8WqVMBaSob4BeFIuHiVKpYCJNv7LR6cI'

    Create custom traits
    curl -i "http://172.18.22.222/placement/traits/CUSTOM_FANGUIJU_HOST" \
    -X PUT \
    -H 'Content-type: application/json' \
    -H 'Accept: application/json' \
    -H 'X-Auth-Project-Id: admin' \
    -H 'OpenStack-API-Version: placement 1.21' \
    -H 'X-Auth-Token:gAAAAABcC12qN3GdLvjYXSSUODi7Dg9jTHUfcnF7I_ljmcffZjs3ignipGLj6iqDvDJ1gXkzGIDW6rRRNcXary-wPfgsb3nCWRIEiAS8LrReI4SYL1KfQiGW7j92b6zTz7RoSEBXACQ9z7UUVfeJ06n8WqVMBaSob4BeFIuHiVKpYCJNv7LR6cI'
    NOTE：自定义 traits 建议以 CUSTOM_ 开头。

    Update all traits for a specific resource provider
    curl -i "http://172.18.22.222/placement/resource_providers/5c5a578f-51b0-481c-b38c-7aaa3394e585/traits" \
    -X PUT \
    -H 'Content-type: application/json' \
    -H 'Accept: application/json' \
    -H 'X-Auth-Project-Id: admin' \
    -H 'OpenStack-API-Version: placement 1.21' \
    -H 'X-Auth-Token:gAAAAABcC12qN3GdLvjYXSSUODi7Dg9jTHUfcnF7I_ljmcffZjs3ignipGLj6iqDvDJ1gXkzGIDW6rRRNcXary-wPfgsb3nCWRIEiAS8LrReI4SYL1KfQiGW7j92b6zTz7RoSEBXACQ9z7UUVfeJ06n8WqVMBaSob4BeFIuHiVKpYCJNv7LR6cI' \
    -d '{"resource_provider_generation": 28, "traits": ["HW_CPU_X86_SSE2", "HW_CPU_X86_SSE", "HW_CPU_X86_MMX", "HW_CPU_X86_SVM", "CUSTOM_FANGUIJU_HOST"]}'

    Return all traits associated with a specific resource provider
    [root@control01 ~]# openstack resource provider trait list 5c5a578f-51b0-481c-b38c-7aaa3394e585
    +----------------------+
    | name                 |
    +----------------------+
    | HW_CPU_X86_SSE2      |
    | HW_CPU_X86_SSE       |
    | HW_CPU_X86_MMX       |
    | HW_CPU_X86_SVM       |
    | CUSTOM_FANGUIJU_HOST |
    +----------------------+


List allocation candidates filter by traits

    # REQ
    curl -i "http://172.18.22.222/placement/allocation_candidates?resources=VCPU:1,MEMORY_MB:512,DISK_GB:5&member_of=5eea7084-0207-44f0-bbeb-c759e8c766a1&required=CUSTOM_FANGUIJU_HOST" \
    -X GET \
    -H 'Content-type: application/json' \
    -H 'Accept: application/json' \
    -H 'X-Auth-Project-Id: admin' \
    -H 'OpenStack-API-Version: placement 1.21' \
    -H 'X-Auth-Token:gAAAAABcC12qN3GdLvjYXSSUODi7Dg9jTHUfcnF7I_ljmcffZjs3ignipGLj6iqDvDJ1gXkzGIDW6rRRNcXary-wPfgsb3nCWRIEiAS8LrReI4SYL1KfQiGW7j92b6zTz7RoSEBXACQ9z7UUVfeJ06n8WqVMBaSob4BeFIuHiVKpYCJNv7LR6cI'

    # RESP
    {
        "provider_summaries": {
            "5c5a578f-51b0-481c-b38c-7aaa3394e585": {
                "traits": ["HW_CPU_X86_SSE2", "HW_CPU_X86_SSE", "HW_CPU_X86_MMX", "HW_CPU_X86_SVM", "CUSTOM_FANGUIJU_HOST"],
                "resources": {
                    "VCPU": {
                        "used": 5,
                        "capacity": 512
                    },
                    "MEMORY_MB": {
                        "used": 3648,
                        "capacity": 60670
                    },
                    "DISK_GB": {
                        "used": 7,
                        "capacity": 49
                    }
                }
            }
        },
        "allocation_requests": [{
            "allocations": {
                "5c5a578f-51b0-481c-b38c-7aaa3394e585": {
                    "resources": {
                        "VCPU": 1,
                        "MEMORY_MB": 512,
                        "DISK_GB": 5
                    }
                }
            }
        }]
    }


# Summarize
The project is still relatively young, and many functions have not been fully implemented. The Stein version has become an independent project for the first time. Although placement only replaces several commonly used filters of nova-scheduler's CpuFilter, CoreFilter and DiskFilter, it is expected that it will eventually replace nova in the future. -scheduler service and can conveniently use, manage, and monitor the system resources of the entire OpenStack.


