**INTEGRATION TESTING (Alloyd Omni \+ OSS components):**

The following steps describe the deployment and testing process:

1. **Manual Execution of Guitar Command:** The process begins with the manual execution of the Guitar Command.  
2. **Triggering the LuSTi Framework:** Guitar subsequently triggers the LuSTi framework.  
3. **VM Provisioning:** LuSTi initiates the creation of necessary Virtual Machines (VMs) based on the defined architecture.  
4. **Infrastructure Setup and Installation:** Setup Playbooks are then executed on the provisioned VMs. These playbooks handle the complete installation of both Omni RPMs and all required OSS components.  
5. **Testing Activation:** Finally, once the infrastructure is installed, the Test Go file is activated to perform validation and testing.

**End-to-End Testing Improvement:**

Migrate from manual execution of the Guitar Command to automated execution via **GUITAR CI**. This change will ensure the Guitar Command is triggered every four hours, utilizing the latest code from the CS head.

**Standalone:  Alloydb cluster create & Destroy:**

**CL:** [https\://critique.corp.google.com/cl/831345542](https://critique.corp.google.com/cl/831345542)  
**Command:** 

```
guitar run -w //storage/alloydb/admin/testing/guitar/nova/integration_tests:omni_nova_integration_tests --version=citc:LOCAL --cluster=alloydb-manual --env_param="omni_rpm_version=16.8.0-7.rhel9" --env_param="rpm_repo_url=https://us-central1-yum.pkg.dev/projects/alloydb-omni-sandbox/nova" --env_param="ma_rpm_version=0.3.0-1.rhel9" --env_param="gpg_check=true" --env_param="vm_zone=us-central1-a" --detach
```

**Fusion Link:** [https\://fusion2.corp.google.com/ci/guitar/workflows/%2F%2Fstorage%2Falloydb%2...](https://fusion2.corp.google.com/ci/guitar/workflows/%2F%2Fstorage%2Falloydb%2Fadmin%2Ftesting%2Fguitar%2Fnova%2Fintegration_tests:omni_nova_integration_tests/activity/11aff316-56e2-30b2-88b8-abcd792b2731:0/invocations/106dfa5c-83da-4710-8e5a-42e9108557ae/targets/%2F%2Fstorage%2Ftesting%2Flusti%2Ftests%2Fnova%2Fintegration_tests%2Fnative:nova_standalone_create_destroy_test/log)

**Next Steps for HA AlloydB Create/Destroy:**

The following playbooks need to be developed:

1. Configure ETCD.  
2. Configure Patroni.  
3. Configure Keepalived.  
4. Develop a Test Playbook.  
5. Develop the Go Lang CL (Command Line).

