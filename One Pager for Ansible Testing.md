# Ansible Testing: Infrastructure Modifications

# Existing Test Automation Infrastructure:

[Add infrastructure code for NOVA automated tests - Standalone cluster type](https://critique.corp.google.com/cl/814644717)

# Problem Statements:

1. I discussed the current Test automation infrastructure with [Arun Kumar J D](mailto:arjd@google.com). The existing test automation infrastructure pre-installs some resources on database and external nodes. This is problematic for Ansible testing, as Ansible collections should handle all installations. In the future, if a component isn't installed via an Ansible role, the current setup will not be able to catch that issue because it's installed by the infrastructure code.   
     
   \-\> [Arun Kumar J D](mailto:arjd@google.com) will discuss how to differentiate the infrastructure code for Test Automation and Ansible orchestrator testing with Prateek and the team.

2. To install Ansible collections, the Ansible team uses default group names in the [inventory file](https://source.corp.google.com/piper///depot/google3/storage/alloydb/nova/automation/ansible/spec/collections/samples/inventory/tests/scalable_ha_with_readpool_with_pgbouncer/inventory.yml) and that doesn’t match with the Ansible group name setup by the infrastructure code. Therefore, we need to modify the group names within the infrastructure code to be compatible with the Ansible orchestrator’s inventory file. As a part of this change, it should also be handled that Test Automation tests are not broken because of this change.

# Conclusion:

We are currently planning to utilize the same infrastructure for test automation and Ansible orchestrator testing. Hence, we need to modify the inventory group names in the infrastructure code to accommodate  Ansible orchestrator testing and Test Automation. [Satya Sai Prabhu Dinesh Sontenam (xWF)](mailto:satyasais@google.com) will be raising a CL with modifications of group names in the inventory setup which is compatible for both test automation and Ansible testing. 