## Intro to LuSTI & Guitar for Database Testing

This document provides a quick overview of Guitar and LuSTI, two key tools we use for integration testing, especially for services interacting with Cloud SQL (Speckle) and AlloyDB (Lux).

### What is Guitar? (The Test Runner)

Think of Guitar ([go/guitar](https://goto.google.com/guitar)) as a powerful system for running tests that are too big, too long, or too complex for standard systems like TAP and Forge.

* **Why Guitar?** We need it when tests:  
  * Need more resources (CPU, RAM, time) than Forge allows.  
  * Must interact with live environments (like staging, autopush, or sandboxes).  
  * Require special test setups (Systems Under Test \- SUTs).  
  * Need to run on specific hardware or Borg jobs.  
* **Key Concepts:**  
  * **Guitar Workflow:** Defined in a `BUILD` file, this tells Guitar *what* to build, *how* to set up any environment, and *which* test targets to run.  
  * **Guitar Cluster:** A pool of Borg jobs dedicated to running Guitar workflows. We have our own team cluster(s) for this.  
* **In Short:** Guitar is the engine that executes our large-scale integration tests.

### What is LuSTI? (The Database Test Framework)

LuSTI ([go/lusti](https://goto.google.com/lusti)) stands for **Lux \+ Speckle Testing Infrastructure**. It's a framework specifically designed to make writing integration tests against **AlloyDB (Lux)** and **Cloud SQL (Speckle)** much simpler and more concise.

* **Why LuSTI?**  
  * **Reduces Boilerplate:** Writing tests that create, configure, and interact with database instances can involve a lot of repetitive code. LuSTI provides high-level Go functions to handle this.  
  * **"Batteries Included":** Comes with built-in helpers and validations for common database testing operations.  
  * **Maintainability:** Standardizes how database tests are written, making them easier to read and maintain.  
  * **Focus on Logic:** Lets us focus on testing our feature's logic rather than the test setup mechanics.  
* **In Short:** LuSTI helps us *write* database integration tests more efficiently.

### How LuSTI and Guitar Work Together

It's a simple partnership:

1. We write our test logic using the **LuSTI** framework in Go. This test code knows how to talk to AlloyDB/Cloud SQL to perform actions and validations.  
2. We define a **Guitar Workflow** in a `BUILD` file. This workflow points to our LuSTI test target(s).  
3. **Guitar** takes this workflow and runs the LuSTI test on a designated **Guitar Cluster**. The test code then executes, interacting with the necessary database environments.

Guitar ExecutionTest ArtifactsSystem Under TestGuitar Workflow EngineGuitar Cluster (Borg Jobs)Schedules test onLuSTI Test Code (Go)//path/to:my\_lusti\_testExecutesRuns onCloud SQL / AlloyDB(Sandbox, Autopush, Staging)Interacts withRapid (Release)Triggers WorkflowBUILD File(Defines Guitar Workflow & LuSTI Test)ConfiguresDefines

### Why Should You Care?

* **Reliability:** These tests help us catch integration bugs between our services and the database backends early.  
* **Automation:** They are run automatically (e.g., as part of presubmits or releases via Rapid), ensuring continuous quality.  
* **Efficiency:** LuSTI makes writing and maintaining these crucial tests much less painful.

### Key Terms Recap

* **Guitar Workflow:** The recipe for what to run (`BUILD` file).  
* **Guitar Cluster:** The workers that run the tests (Borg jobs).  
* **LuSTI Test:** The actual Go test code using the LuSTI framework (e.g., `create_test.go`).  
* **SUT (System Under Test):** The environment the test targets (e.g., an AlloyDB sandbox).

**Conceptual Flow:**

1. **Rapid:** Your release process orchestrator.  
2. **Rapid Workflow:** A specific set of tasks within Rapid (defined in a `.pp` file).  
3. **Guitar Task:** A task within your Rapid Workflow that tells Guitar to run something.  
4. **Guitar Workflow:** A definition (in a `BUILD` file) telling Guitar *which* tests to run and *how*.  
5. **LuSTI Test:** Your actual test code (Go test, using LuSTI framework), defined as a target in a `BUILD` file. This test likely interacts with Cloud SQL or AlloyDB environments.  
6. **Guitar Cluster:** The set of Borg jobs that actually execute the Guitar Workflow.

Here's a step-by-step guide:

**Step 1: Have a LuSTI Test Target**

You need an existing LuSTI test. Let's assume you have one defined in `//path/to/my/lusti/tests/BUILD`. Example from [go/lusti-test-overview](https://goto.google.com/lusti-test-overview):

PYTHON

`# //path/to/my/lusti/tests/BUILD`

`load("//storage/testing/lusti/tests/create:create.bzl", "lusti_create_test")`  
`# ... other loads`

`# Example LuSTI test definition`  
`lusti_create_test(`  
    `name = "my_demo_lusti_create_test",`  
    `# ... other lusti specific args like 'database', 'envs', 'owners', etc.`  
`)`

This creates test targets like `//path/to/my/lusti/tests:my_demo_lusti_create_test_integration_e2e_env_in_a_cluster_no_env_deps` (the exact name depends on the LuSTI macro and environment). Find the exact target name using `blaze query //path/to/my/lusti/tests:all`.

**Step 2: Define a Guitar Workflow to Run the LuSTI Test**

Create a `guitar_workflow_test` rule in a `BUILD` file (e.g., `//path/to/my/guitar/BUILD`). This workflow will wrap your LuSTI test target.

PYTHON

`# //path/to/my/guitar/BUILD`  
`load("//testing/integration/guitar/build_defs:guitar_workflow.bzl", "guitar", "guitar_workflow_test")`

`guitar_workflow_test(`  
    `name = "my_lusti_demo_guitar_workflow",`  
    `integration_test = guitar.IntegrationTest(`  
        `tests = [`  
            `guitar.Tests(`  
                `execution_method = "DISTRIBUTED_ON_BORG",  # LuSTI tests usually run on Borg`  
                `targets = [`  
                    `# Replace with your actual LuSTI test target`  
                    `"//path/to/my/lusti/tests:my_demo_lusti_create_test_integration_e2e_env_in_a_cluster_no_env_deps",`  
                `],`  
                `# Pass arguments to the LuSTI test if needed`  
                `args = [`  
                    `"--test_environment=autopush",  # Example: target a specific env`  
                    `# "--existing_sandbox=alloydbadmin-{REQUESTER}-sandbox", # If using sandboxes`  
                    `"--noteardown_sandbox",`  
                `],`  
                `per_target_time_limit_secs = 7200,  # Adjust as needed`  
            `),`  
        `],`  
        `# Add env_params if you need to parameterize the workflow from Rapid`  
        `# env_params = {`  
        `#     "my_param": "default_value",`  
        `# },`  
    `),`  
    `tags = ["manual"],`  
`)`

* **`execution_method`**: Since LuSTI tests interact with external database instances, they can't run on `FORGE`. You'll typically use `DISTRIBUTED_ON_BORG`, which requires a Guitar cluster.  
* **`targets`**: List the specific LuSTI test target(s) you want to run.  
* **`args`**: Pass any necessary flags to your LuSTI test. See [go/lusti-devguide-running](https://goto.google.com/lusti-devguide-running) for common flags.

**Step 3: Test the Guitar Workflow Manually**

Before involving Rapid, make sure you can trigger the Guitar workflow manually. You'll need a Guitar cluster (e.g., `my-team-cluster`). See [go/guitar-cluster](https://goto.google.com/guitar-cluster) for cluster types.

BASH

`# Install Guitar CLI if you haven't: sudo apt install google-guitar`

`# Run the workflow on your team's cluster`  
`guitar run -w //path/to/my/guitar:my_lusti_demo_guitar_workflow \`  
  `--cluster=my-team-cluster \`  
  `-v citc:LOCAL`

`# To run on your local machine (for testing the workflow definition,`  
`# but may fail due to permissions/network for the actual LuSTI test calls)`  
`# See go/guitar-local`  
`# guitar run -w //path/to/my/guitar:my_lusti_demo_guitar_workflow --cluster=LOCAL -v citc:LOCAL`

**Step 4: Create/Update Rapid Workflow Definition (.pp file)**

In your Rapid project's codebase, define or modify a `.pp` file to include the Guitar task.

PATCHPANEL

`// example_rapid_workflow.pp`  
`import '//releasetools/rapid/workflows/rapid.pp' as rapid`

`vars = rapid.create_vars() {}`

`task_properties = [`  
  `'guitar.manual_trigger': [`  
    `'guitar_workflows=//path/to/my/guitar:my_lusti_demo_guitar_workflow',`  
    `'guitar_cluster=my-team-cluster', // *** REPLACE with your Guitar cluster name ***`  
    `'guitar_detach=False', // Wait for Guitar results`  
    `'guitar_error_on_test_failure=True', // Fail Rapid task if tests fail`  
    `// Use the version from the Rapid candidate`  
    `'guitar_versions=cl:CANDIDATE',`  
    `// Example of passing env_params to the Guitar workflow`  
    `// 'guitar_env_params=my_param=value_from_rapid',`  
  `],`  
`]`

`task_deps = [`  
  `'guitar.manual_trigger': ['start'],`  
  `// Add other tasks as needed`  
`]`

`workflow run_lusti_tests = rapid.workflow([task_deps, task_properties]) {`  
  `vars = @vars`  
`}`

* See [go/guitar-rapid-plugin](https://goto.google.com/guitar-rapid-plugin) for all `guitar.manual_trigger` options.  
* **`guitar_cluster`**: This *must* be specified and be a cluster your Rapid runner has permissions to use.

**Step 5: Update Your Blueprint File**

Ensure your blueprint file (e.g., `my_rapid_project.blueprint`) includes this `.pp` file as a custom workflow.

NCL

`// my_rapid_project.blueprint`  
`include "devtools/blueprint/ncl/blueprint_file.ncl";`  
`include "releasetools/rapid/ncl/rapid_config.ncl";`

`blueprint_file = ::blueprint::BlueprintFile(`  
  `// ... other blueprint settings (project_name, teams_product_id, etc.)`

  `buildable_units = [`  
    `// ... your other buildable units`  
    `::blueprint::BuildableUnit(`  
      `name = "my_lusti_guitar_workflows",`  
      `test_patterns = [`  
        `"//path/to/my/guitar:my_lusti_demo_guitar_workflow",`  
      `],`  
      `enable_release = false, // Typically false, Guitar BUs are not for packaging`  
    `),`  
  `],`

  `releasable_units = [`  
    `::blueprint::ReleasableUnit(`  
      `name = "my_rapid_project",`  
      `// ...`

      `rapid_config = ::Rapid::RapidConfig(`  
        `// ... grant_settings, runner_config, etc.`

        `workflows = [`  
          `// ... other workflows`  
          `::Rapid::Workflow::CustomWorkflow(`  
            `name = "Run LuSTI Integration Tests",`  
            `config_path = "google3/path/to/your/pp/files/example_rapid_workflow.pp",`  
            `description = "Runs the demo LuSTI tests on Guitar",`  
          `),`  
        `],`

        `// Optional: Automate this workflow`  
        `// automation_rule = [ ... ]`  
      `),`  
    `),`  
  `],`  
`);`

**Step 6: Test in Rapid**

1. Submit the changes to your BUILD, .pp, and blueprint files.  
2. Go to your Rapid project in the Rapid UI.  
3. Create a new Release and Candidate.  
4. Once the Candidate is created, click "Launch Workflow" and select "Run LuSTI Integration Tests".  
5. Monitor the workflow execution. Click the "Trigger\[guitar.manual\_trigger\]" task logs to find a link to the Guitar results (Fusion).

**Important Considerations:**

* **Guitar Cluster Permissions:** The user account under which your Rapid runners operate (e.g., `my-project-releaser`) needs to be in the ACLs of the Guitar cluster (`my-team-cluster`) to be able to trigger workflows.  
* **LuSTI Test Environment:** LuSTI tests often require a specific environment (e.g., a Cloud SQL or AlloyDB sandbox, or a static realm like autopush/staging). Ensure the environment is accessible from the Guitar cluster workers. You might need to pass environment details (like sandbox names or endpoints) as `args` or `guitar_env_params`.  
* **Authentication:** The tests running on the Guitar cluster workers will use the worker's service account credentials to interact with GCP services. Ensure this account has the necessary IAM permissions on your test projects.  
* **Rapid QA:** For testing changes to blueprints and workflow definitions without affecting production Rapid, consider using [go/rapid-qa](https://goto.google.com/rapid-qa).  
* 

**Documented:**   
In context of LuSTI (Lux \+ Speckle Testing Infrastructure), "proto" refers to **Protocol Buffers**.

1. **What Protocol Buffers Are:**  
   * Protocol Buffers (or Protos) are a language-neutral, platform-neutral, extensible mechanism developed at Google for serializing structured data. Think of them like XML or JSON, but typically smaller, faster, and simpler.  
   * You define how you want your data to be structured once in a `.proto` file.  
   * The proto compiler generates code in various languages (like Go, Java, Python, C++) to easily write and read the structured data to and from various data streams.  
   * They are used extensively across Google for data storage, RPCs (like Stubby/gRPC), and configuration.  
2. **How LuSTI Uses Protos for Configuration:**  
   * LuSTI uses Protocol Buffers to manage test configurations in a structured way, instead of relying on a multitude of command-line flags. This makes configurations easier to write, read, and maintain.  
   * The core configuration definitions are located in: `google3/storage/testing/lusti/src/config.proto`.  
   * The main message is `Flags`. It has two key parts:  
     * `SharedFlags shared`: Contains settings common to *all* LuSTI tests (e.g., project, region).  
     * `oneof TestCaseFlags`: Contains messages specific to *each type* of test. Each field in the `oneof` is a different message type, corresponding to a particular test's needs.  
   * **Example from your files:**  
     * In `default_cloudbuild_test.go`, you see `confpb "google3/storage/testing/lusti/src/config_go_proto"`.  
     * The test function `lusti.Run` takes a callback: `func(t *lusti.Test, flags *confpb.CloudbuildTest)`. This means this specific test expects its configuration to be provided as a `CloudbuildTest` message, which is one of the types defined within the `TestCaseFlags` `oneof` in `config.proto`.  
     * The `CloudbuildTest` message likely defines fields such as `cloudbuild_yaml_path`, `cloudbuild_source_path`, details about the SMURF instance to create (`smurf_instance`), etc., as used in `cloudbuild_test_runner.go`.  
   * **How Configuration is Provided:**  
     * While the *definition* is in `.proto` files, the *actual values* for a specific test run are often constructed within BUILD files using Starlark macros (`.bzl` files).  
     * These macros generate a textproto representation of the `Flags` message (including the relevant `TestCaseFlags` variant like `CloudbuildTest`).  
     * This textproto is passed to the test binary at runtime, usually via a flag, and LuSTI's `lusti.NewConfigFromFlags` parses it into the Go struct representations.

In summary, LuSTI leverages Protocol Buffers to provide a robust and flexible way to configure different tests, keeping shared parameters separate from test-specific ones.

**Simple terminology:**

Imagine you're giving instructions to someone to bake a cake. You could shout out ingredients and steps one by one, but it's easy to miss something or get things in the wrong order.

**Proto (Protocol Buffer) is like a super organized Recipe Card Template.**

1. **The Template Definition (.proto file):**  
   * Someone at Google created a blank recipe card template called "Protocol Buffer."  
   * The LuSTI team then designed a specific set of fill-in-the-blanks on this template *just for configuring tests*. This design is in files ending with `.proto`, like `config.proto`.  
   * This template has sections:  
     * **Common Info:** Stuff needed by *almost all* tests (e.g., "Which GCP Project?", "Which Region?"). This is like the top part of the recipe card for any cake.  
     * **Specific Test Instructions:** Sections that only apply to *certain types* of tests. For example:  
       * If it's a "Create a database test," there are blanks for "Database Version?", "Instance Size?".  
       * If it's your "Cloud Build test," there are blanks for "Path to YAML file?", "Path to source code?".  
       * If it's a different test type, it has its *own* set of blanks.  
2. **Filling in the Recipe Card (BUILD / .bzl files):**  
   * When you want to run a specific test, you don't use command-line flags for *every little detail*.  
   * Instead, the BUILD files use helper functions (macros in `.bzl` files) to take your high-level requests and fill out a copy of that LuSTI recipe card template. This filled-out card is often represented as a "textproto".  
   * This textproto has all the settings for *that specific run*.  
3. **The Test Using the Filled Card (Go code):**  
   * The actual LuSTI test code (like `default_cloudbuild_test.go`) is written to expect a filled-in recipe card.  
   * When the test starts, LuSTI reads the filled-in textproto.  
   * The Go code then looks at the values in the blanks to know exactly what to do (e.g., "Aha\! I need to use *this* GCP project, create a VM, and then run the Cloud Build job defined in *this* YAML file.").

**Why is this good?**

In LuSTI, "flags" can refer to a couple of things, but the main configuration is done through Protocol Buffers rather than many individual command-line flags.

Here's the breakdown:

1. **Primary Configuration: Protocol Buffers (Protos)**  
   * Instead of requiring you to set many command-line flags for every test detail, LuSTI tests are primarily configured using a Protocol Buffer message. The main message is `Flags` defined in `google3/storage/testing/lusti/src/config.proto`.  
   * This `Flags` message has a `shared` part for settings common to all tests (like project, region) and a `oneof` part (`TestCaseFlags`) for settings specific to the type of test being run (e.g., `CreateTestFlags`, `CloudbuildTest`).  
   * The actual configuration values for a specific test run are typically defined within BUILD files using Starlark macros (`.bzl` files). These macros generate a **textproto** string representing the filled-in `Flags` message.  
2. **Command-Line Flags:** While the bulk of the configuration is in the proto, some command-line flags are still used:  
   * **To Pass the Proto:** A flag is used to pass the generated textproto configuration to the test binary. The `lusti.NewConfigFromFlags` function in your Go test code (like in `default_cloudbuild_test.go`) is responsible for parsing this.  
   * **Framework Flags:** LuSTI has a few of its own command-line flags to control the test environment and framework behavior. These are often used to set or override values within the `SharedFlags` part of the configuration proto. Examples include:  
     * `--lusti_gcp_project`: Specifies the GCP project.  
     * `--lusti_region`: Specifies the GCP region.  
     * `--test_environment`: Specifies the target environment (e.g., `autopush`, `staging`, `sandbox`). (See [go/lusti-devguide-running](https://goto.google.com/lusti-devguide-running))  
     * `--existing_sandbox`: Specifies the name of an existing sandbox to use.  
     * `--noteardown_sandbox`: Prevents tearing down the sandbox after the test.  
     * Logging flags like `--vmodule`.  
   * **Blaze Test Arguments:** When you run a LuSTI test using `blaze test`, you pass these command-line flags to the *test binary* using the `--test_arg` prefix. For example:  
   * BASH

`blaze test //path/to:my_lusti_test \`  
  `--notest_loasd \`  
  `--test_arg=--lusti_gcp_project=my-project-id \`  
  `--test_arg=--test_environment=autopush`

* The `--notest_loasd` flag is also crucial for LuSTI tests to use your real credentials.

In essence, LuSTI uses flags to bootstrap the test and set up the environment, but the detailed configuration for *what* the test should do is encapsulated within the proto message, which is generated by the build system. This approach keeps test definitions structured and avoids an explosion of command-line arguments.

**Goal:** The codelab walks you through creating a LuSTI test that sets up an AlloyDB cluster and then performs a random sequence of operations (like failover, resizing a read pool, or setting database flags) on it.

**Basic Perspective:**

* **LuSTI (Lux \+ Speckle Testing Infrastructure):** A framework designed to simplify writing integration tests for AlloyDB (Lux) and Cloud SQL (Speckle). It provides higher-level abstractions for common tasks like creating instances, performing operations, and validating state. See [go/lusti](https://goto.google.com/lusti).  
* **Configuration over Code:** LuSTI tests are heavily driven by configuration defined in a `.proto` file. Instead of writing lots of Go code for setup, you define the desired state and test parameters in a textproto, which the LuSTI framework parses.  
* **Guitar:** A framework for running integration tests, often used for tests that are too large or take too long for TAP/Forge. LuSTI tests are typically run using Guitar. See [go/guitar](https://goto.google.com/guitar).  
* **Sandbox:** Tests like these don't run against production. They usually run in a sandbox environment (e.g., using Centigrate), which is an isolated instance of the AlloyDB control plane and related dependencies.

Here are the steps to create and run the codelab test:

**Create a CitC Client:**

1. Open a terminal and create a new CitC client:  
2. BASH

`g4d -f lusti_codelab`

3. 

**Modify config.proto:**  
Open `google3/storage/testing/lusti/src/config.proto`.  
a. Add `RandomOpTest` to the `TestCaseFlags` oneof within the `Flags` message:  
`protobuf message Flags { // ... existing fields ... oneof TestCaseFlags { // ... existing test flags ... RandomOpTest random_op_test = 14; // Add this line } }`  
b. Add the `RandomOpTest` message definition at the end of the file:

4. `protobuf message RandomOpTest { message ReadPoolResize { string pool_name = 1; int32 new_node_count = 2; } message Failover {} message SetFlags { string pool_name = 1; map<string, string> database_flags = 2; } message Op { int32 weight = 1; google.protobuf.Duration timeout = 2; oneof operation { ReadPoolResize read_pool_resize = 3; Failover failover = 4; SetFlags set_flags = 5; } } LuxCluster instance = 1; repeated Op ops = 2; int32 count = 3; int64 seed = 4; }`

**Create the Test File:**

5. Create the file `google3/storage/testing/lusti/tests/random/random_test.go` with the following content:  
6. GO

`package random_test`

`import (`  
	`"context"`  
	`"fmt"`  
	`"math/rand"`  
	`"sort"`  
	`"testing"`  
	`"time"`

	`confpb "google3/storage/testing/lusti/src/config_go_proto"`  
	`"google3/storage/testing/lusti/src/lusti"`  
	`"google3/storage/testing/lusti/src/lux"`  
`)`

`func TestRandomOps(t *testing.T) {`  
	`lt := &lusti.Test{`  
		`T:            t,`  
		`ConfigSource: lusti.NewConfigFromFlags,`  
	`}`  
	`lusti.Run(context.Background(), lt, func(t *lusti.Test, f *confpb.RandomOpTest) {`

		`namer := lux.NewStandardNamer(t.TestID(), "instance")`  
		`cp := t.Lux().NewDBInstance(t, namer, f.GetInstance())`

		`for _, op := range f.GetOps() {`  
			`rp := ""`  
			`switch op.WhichOperation() {`  
			`case confpb.RandomOpTest_Op_ReadPoolResize_case:`  
				`rp = op.GetReadPoolResize().GetPoolName()`  
				`if rp == "" {`  
					`t.Fatalf("Resize Op did not specify a readpool")`  
				`}`  
			`case confpb.RandomOpTest_Op_SetFlags_case:`  
				`rp = op.GetSetFlags().GetPoolName()`  
			`case confpb.RandomOpTest_Op_Failover_case:`  
				`continue`  
			`}`  
			`if rp != "" {`  
				`if _, ok := cp.ReadPools[rp]; !ok {`  
					`t.Fatalf("Read Pool %q does not exist.", rp)`  
				`}`  
			`}`  
		`}`

		`cs := lux.ConnectionSettings{}`  
		`cluster := t.Lux().CreateCluster(t, cp, cs)`  
		`cluster.Primary.LoadDataset(t, "letters", cs)`

		`var weights []int`  
		`var sum int = 0`  
		`for _, op := range f.GetOps() {`  
			`sum += int(op.GetWeight())`  
			`weights = append(weights, sum)`  
		`}`  
		`r := rand.New(rand.NewSource(f.GetSeed()))`  
		`for i := int32(0); i < f.GetCount(); i++ {`  
			`rn := r.Intn(sum)`  
			`opi := sort.SearchInts(weights, rn)`  
			`op := f.GetOps()[opi]`  
			`timeout := time.Duration(op.GetTimeout().GetSeconds())*time.Second +`  
				`time.Duration(op.GetTimeout().GetNanos())*time.Nanosecond`  
			`switch op.WhichOperation() {`  
			`case confpb.RandomOpTest_Op_Failover_case:`  
				`t.Run("Failover", timeout, func(t *lusti.Test) {`  
					`cluster.Primary.Failover(t, cs)`  
				`})`  
			`case confpb.RandomOpTest_Op_SetFlags_case:`  
				`sfop := op.GetSetFlags()`  
				`inst := cluster.Primary.Instance`  
				`name := "Set Primary Flags"`  
				`if sfop.GetPoolName() != "" {`  
					`inst = cluster.ReadPools[sfop.GetPoolName()].Instance`  
					`name = fmt.Sprintf("Set %s Flags", sfop.GetPoolName())`  
				`}`  
				`t.Run(name, timeout, func(t *lusti.Test) {`  
					`inst.SetFlagsTo(t, sfop.GetDatabaseFlags(), nil, cs)`  
				`})`  
			`case confpb.RandomOpTest_Op_ReadPoolResize_case:`  
				`rop := op.GetReadPoolResize()`  
				`t.Run("Resize "+rop.GetPoolName(), timeout, func(t *lusti.Test) {`  
					`inst := cluster.ReadPools[rop.GetPoolName()]`  
					`inst.UpdateNodeCount(t, int(rop.GetNewNodeCount()), cs)`  
				`})`  
			`}`  
		`}`  
	`})`  
`}`

7. 

**Create the BZL Macro File:**

8. Create `google3/storage/testing/lusti/tests/random/random.bzl` with this content:  
9. BZL

`"""Contains macros for writing tests for performing random operation on an instance"""`

`load(`  
    `"//storage/testing/lusti/src:lusti.bzl",`  
    `"TYPE_LUX",`  
    `"default_vars",`  
    `"lusti_multi_env_variant",`  
    `"lusti_test_settings",`  
    `"proto_list",`  
    `"proto_map",`  
    `"shared_flags",`  
`)`

`def lusti_random_op_test(`  
        `name,`  
        `owners,`  
        `database,`  
        `ops,`  
        `count,`  
        `seed,`  
        `test_id,`  
        `envs,`  
        `base_test,`  
        `test_settings = lusti_test_settings(),`  
        `**kwargs):`  
    `"""Generates a LuSTI test that will randomly perform operations on a cluster.`

    `Args:`  
        `name: name of the generated test.`  
        `owners: owners of this test. See go/lusti-devguide-ownership.`  
        `database: db  instance to create.`  
        `ops: a list of operations to perform.`  
        `count: the number of operations to perform.`  
        `seed: initial seed for th random number genrator.`  
        `test_id: Used to generate keys that are unique to this test case.`  
        `base_test: the base test target to pass arguments to. Should be generated by lusti_centigrate_go_test().`  
        `envs: A test_envs() struct that controls which environments tests get created for.`  
        `test_settings: settings related to test execution`  
        `**kwargs: args to pass through to the go_test rule`  
    `Returns:`  
        `centgriate test rule`  
    `"""`  
    `if database.type != TYPE_LUX:`  
        `fail("Random test only supports lux instances")`  
    `shared = shared_flags(True, test_id, test_settings, vars = default_vars(envs))`

    `textproto = """shared: {{`  
`{shared}`  
`}}`  
`random_op_test:{{`  
  `instance: {{`  
`{instance}`  
  `}}`  
  `ops: {ops}`  
  `count: {count}`  
  `seed: {seed}`  
`}}`  
`""".format(`  
        `shared = shared,`  
        `instance = database.instance,`  
        `ops = proto_list(ops),`  
        `count = count,`  
        `seed = seed,`  
    `)`  
    `return lusti_multi_env_variant(`  
        `name = name,`  
        `envs = envs,`  
        `base_test = base_test,`  
        `textproto = textproto,`  
        `owners = owners,`  
        `tags = [database.type],`  
        `**kwargs`  
    `)`

`def read_pool_resize_op(`  
        `weight,`  
        `timeout_sec,`  
        `pool_name,`  
        `new_node_count):`  
    `"""Generates an op that will resize a read pool.`

    `Args:`  
        `weight: the relative odds that should occur compared to other oerations.`  
        `timeout_sec: how long to wait for this operation to finish in seconds.`  
        `pool_name: name of pool to resize.`  
        `new_node_count: the new number of nodes for the read pool.`  
    `Returns:`  
        `An op text proto`  
    `"""`  
    `return """`  
    `weight: {weight}`  
    `timeout: {{`  
      `seconds: {timeout_sec}`  
    `}}`  
    `read_pool_resize: {{`  
      `pool_name: "{pool_name}"`  
      `new_node_count: {new_node_count}`  
    `}}`  
    `""".format(`  
        `weight = weight,`  
        `timeout_sec = timeout_sec,`  
        `pool_name = pool_name,`  
        `new_node_count = new_node_count,`  
    `)`

`def failover_op(`  
        `weight,`  
        `timeout_sec):`  
    `"""Generates an op that will resize a failover the cluster primary instance.`

    `Args:`  
        `weight: the relative odds that should occur compared to other oerations.`  
        `timeout_sec: how long to wait for this operation to finish in seconds.`  
    `Returns:`  
        `An op text proto`  
    `"""`  
    `return """`  
    `weight: {weight}`  
    `timeout: {{`  
      `seconds: {timeout_sec}`  
    `}}`  
    `failover: {{}}`  
    `""".format(`  
        `weight = weight,`  
        `timeout_sec = timeout_sec,`  
    `)`

`def set_flags_op(`  
        `weight,`  
        `timeout_sec,`  
        `database_flags,`  
        `pool_name = ""):`  
    `"""Generates an op that will set the flags of an instance.`

    `Args:`  
        `weight: the relative odds that should occur compared to other oerations.`  
        `timeout_sec: how long to wait for this operation to finish in seconds.`  
        `pool_name: name of pool to resize. If unset the cluster primary instance will be used.`  
        `database_flags: the flags that the instance will have set.`  
    `Returns:`  
        `An op text proto`  
    `"""`  
    `return """`  
    `weight: {weight}`  
    `timeout: {{`  
      `seconds: {timeout_sec}`  
    `}}`  
    `set_flags: {{`  
      `pool_name: "{pool_name}"`  
      `database_flags: {database_flags}`  
    `}}`  
    `""".format(`  
        `weight = weight,`  
        `timeout_sec = timeout_sec,`  
        `pool_name = pool_name,`  
        `database_flags = proto_map(database_flags),`  
    `)`

10. 

**Create the BUILD File:**

11. Create `google3/storage/testing/lusti/tests/random/BUILD` with this content:  
12. BZL

`load("//tools/build_defs/testing:bzl_library.bzl", "bzl_library")`  
`load("//storage/alloydb/admin/testing/centigrate:locations.bzl", "PRIMARY_REGION")`  
`load("//storage/alloydb/admin/testing/centigrate:sandboxes.bzl", "sandbox")`  
`load(`  
    `"//storage/testing/lusti/tests/random:random.bzl",`  
    `"failover_op",`  
    `"lusti_random_op_test",`  
    `"read_pool_resize_op",`  
    `"set_flags_op",`  
`)`  
`load(`  
    `"//storage/testing/lusti/src:lusti.bzl",`  
    `"lusti_multi_env_go_test",`  
    `"lux_database",`  
    `"lux_read_pool",`  
    `"test_envs",`  
`)`

`_ENVS = test_envs(`  
    `sandbox_envs = [`  
        `sandbox.alloydb.default([PRIMARY_REGION]),`  
    `],`  
`)`

`lusti_multi_env_go_test(`  
    `name = "random_test",`  
    `size = "large",`  
    `timeout = "eternal",`  
    `srcs = ["random_test.go"],`  
    `envs = _ENVS,`  
    `glaze_kind = "go_test",`  
    `deps = [`  
        `"//storage/testing/lusti/src:config_go_proto",`  
        `"//storage/testing/lusti/src:lusti",`  
        `"//storage/testing/lusti/src:lux",`  
    `],`  
`)`

`lusti_random_op_test(`  
    `name = "lusti_lux_uniform_test",`  
    `size = "large",`  
    `timeout = "eternal",`  
    `base_test = ":random_test",`  
    `count = 3,`  
    `database = lux_database(`  
        `read_pools = {`  
            `"pool": lux_read_pool(`  
                `node_count = 1,`  
            `),`  
        `},`  
        `region = PRIMARY_REGION,`  
    `),`  
    `envs = _ENVS,`  
    `ops = [`  
        `read_pool_resize_op(`  
            `new_node_count = 2,`  
            `pool_name = "pool",`  
            `timeout_sec = 20 * 60,`  
            `weight = 2,`  
        `),`  
        `read_pool_resize_op(`  
            `new_node_count = 1,`  
            `pool_name = "pool",`  
            `timeout_sec = 20 * 60,`  
            `weight = 2,`  
        `),`  
        `failover_op(`  
            `timeout_sec = 10 * 60,`  
            `weight = 4,`  
        `),`  
        `set_flags_op(`  
            `database_flags = {"log_statement_stats": "on"},`  
            `timeout_sec = 20 * 60,`  
            `weight = 1,`  
        `),`  
        `set_flags_op(`  
            `database_flags = {"log_statement_stats": "off"},`  
            `timeout_sec = 20 * 60,`  
            `weight = 1,`  
        `),`  
        `set_flags_op(`  
            `database_flags = {"log_statement_stats": "on"},`  
            `pool_name = "pool",`  
            `timeout_sec = 20 * 60,`  
            `weight = 1,`  
        `),`  
        `set_flags_op(`  
            `database_flags = {"log_statement_stats": "off"},`  
            `pool_name = "pool",`  
            `timeout_sec = 20 * 60,`  
            `weight = 1,`  
        `),`  
    `],`  
    `owners = ["satyasais"],  # Replace with your username`  
    `seed = 5432,`  
    `test_id = "random-000",`  
`)`

`bzl_library(`  
    `name = "random_bzl",`  
    `srcs = ["random.bzl"],`  
    `parse_tests = True,`  
    `visibility = ["//visibility:private"],`  
    `deps = ["//storage/testing/lusti/src:lusti_bzl"],`  
`)`

13. *Make sure to replace `satyasais` with your username in the `owners` field.*

**Run the Test:**  
a. **Start Your Sandbox:** LuSTI tests require a sandbox. Instructions for starting an AlloyDB sandbox are at [go/alloydb-sandbox](https://goto.google.com/alloydb-sandbox). This typically involves a command like:  
`bash source storage/alloydb/admin/testing/centigrate/aliases.sh integration_e2e_env-setup`

14. This command can take a significant amount of time to complete.  
    b. **Execute the Test:** Once the sandbox is running (it will have output similar to `Centigrate environment "alloydbadmin-%USERNAME%-sandbox" is healthy`), run the test using Blaze. Replace `%USERNAME%` with your username if it's not automatically substituted.  
15. CODE BLOCK

```` ```bash ````  
`blaze test \`  
`-c opt \`  
`//storage/testing/lusti/tests/random:lusti_lux_uniform_test_integration_e2e_env_in_a_cluster_no_env_deps \`  
`--notest_loasd \`  
`--test_arg=--existing_sandbox=alloydbadmin-satyasais-sandbox \`  
`--test_arg=--noteardown_sandbox \`  
`'--test_arg=--vmodule=google3/storage/speckle/*=1,google3/storage/speckle/*/*=1,google3/storage/speckle/*/*/*=1,google3/storage/speckle/*/*/*/*=1,google3/storage/speckle/*/*/*/*/*=1,google3/storage/testing/lusti/*=1,google3/storage/testing/lusti/*/*=1,google3/storage/testing/lusti/*/*/*=1' \`  
`--test_output=streamed`  
```` ``` ````  
``**Note:** Ensure the `--existing_sandbox` argument matches your sandbox name (e.g., `alloydbadmin-satyasais-sandbox`).``

16. c. **View Results:** The command will stream output to your console. A Sponge link will be provided at the end, where you can view detailed test logs, subtest results, and Dapper traces as described in the codelab.

For more details on running LuSTI tests, see [go/lusti-devguide-running](https://goto.google.com/lusti-devguide-running).

satyasais@dinesh-sontenam:/google/src/cloud/satyasais\$ g4d \-f lusTi\_codelab  
Creating a new CitC client: 'lusTi\_codelab'  
Currently synced @791558261  
satyasais@dinesh-sontenam:/google/src/cloud/satyasais/lusTi\_codelab/google3\$ 

**LuSTI Overview**

LuSTI (Lux \+ Speckle Testing Infrastructure) is a framework designed to simplify writing integration tests for AlloyDB (Lux) and Cloud SQL (Speckle). Its main goals are:

* **Conciseness:** Provide high-level Go functions for common operations (e.g., create instance, failover, update flags), reducing boilerplate code.  
* **Configuration-Driven:** Use textproto files to define test parameters and environment setup, separating configuration from test logic.  
* **Reusability:** Encourage reusable components and test patterns.  
* **Built-in Validation:** Many LuSTI operations automatically include checks to ensure the system is healthy and the operation completed as expected.

**Necessary Files and Their Roles**

1. **`google3/storage/testing/lusti/src/config.proto`**  
   * **Purpose:** This file defines the structure of configuration data that your test can receive. Think of it as a blueprint for the settings your test needs.  
   * **Key Message:** `Flags`. This is the root message for any LuSTI test configuration. It's passed to the test as a textproto.  
   * **Structure:**  
     * `SharedFlags shared`: Contains settings common to *all* LuSTI tests (e.g., project, environment).  
     * `oneof TestCaseFlags`: Contains messages specific to *each type* of test. Your `RandomOpTest` is added here. This design ensures a test only includes configuration relevant to it.  
   * **Your Addition:** `RandomOpTest random_op_test = 14;` in the `oneof`, and the `message RandomOpTest { ... }` definition itself. This custom message holds all parameters needed for *your specific* random operation test (like the cluster details, operations list, counts, seed).  
2. **`google3/storage/testing/lusti/tests/random/random_test.go`**  
   * **Purpose:** Contains the actual Go test logic.  
   * **Key Components:**  
     * Standard Go test function `TestRandomOps(t *testing.T)`.  
     * Initialization of `lusti.Test`: Wraps the standard `*testing.T` and provides LuSTI-specific context and helpers.  
     * `lusti.NewConfigFromFlags`: Specifies that the test configuration will be read from command-line flags (where the textproto is passed).  
     * `lusti.Run()`: The main entry point for executing the LuSTI test. It handles parsing the configuration, extracting the correct test-specific message (`RandomOpTest` in this case using reflection), and running the provided callback function.  
     * **Test Callback Function:** `func(t *lusti.Test, f *confpb.RandomOpTest)`  
       * `t *lusti.Test`: The LuSTI test context, used for all interactions with the framework and system under test.  
       * `f *confpb.RandomOpTest`: The parsed configuration specific to this test.  
       * **Logic:** Creates an AlloyDB cluster, loads data, and then randomly executes operations as defined in the configuration.  
3. **`google3/storage/testing/lusti/tests/random/random.bzl`**  
   * **Purpose:** Defines custom Starlark (BZL) macros to help generate BUILD targets. This avoids repetitive BUILD file code.   
     	This is a Starlark macro file. Starlark is the language used to write BUILD files. This file defines a reusable function (`lusti_list_rpms_test`) to generate multiple, similar test targets in the `BUILD` file.  
   * **Key Macros:**  
     * `lusti_random_op_test()`: The main macro for this codelab. It takes high-level arguments (like instance details, operation list) and constructs the full textproto configuration string. It then calls `lusti_multi_env_variant` to create the actual test targets.  
     * `read_pool_resize_op()`, `failover_op()`, `set_flags_op()`: Helper macros to make defining individual operations within the `ops` list in the BUILD file more readable. They return textproto snippets for the `Op` message.  
   * **Mechanism:** These macros primarily perform string formatting to build the final textproto string that will be passed as an argument to the Go test.  
4. **`google3/storage/testing/lusti/tests/random/BUILD`**  
   * **Purpose:** This file tells Blaze (Google's build system) how to build and run your tests.  
   * **Key Targets:**  
     * `lusti_multi_env_go_test(name = "random_test", ...)`: Defines the base Go test binary. `glaze_kind = "go_test"` tells Glaze to treat it like a standard Go test for dependency management.  
     * `lusti_random_op_test(name = "lusti_lux_uniform_test", ...)`: Uses the custom macro from `random.bzl` to define a specific test case. This is the target you actually run. It bundles the Go code with the specific configuration generated by the macro.  
     * `bzl_library`: Makes the `random.bzl` file available.

**Key Terms and Concepts**

* **Textproto:** A human-readable text format representing a Protocol Buffer message. LuSTI uses this to pass the extensive configuration to the test binary.  
* **Starlark (BZL):** The configuration language used by Blaze and in BUILD files. Macros in `.bzl` files are functions that generate BUILD rules.  
* **Macro:** A function in a `.bzl` file that can be called from a BUILD file to generate rules. `lusti_random_op_test` is a macro.  
* **Sandbox:** An isolated environment to run tests without affecting production. For AlloyDB, this is typically managed by **Centigrate** ([go/centigrate](https://goto.google.com/centigrate)). The sandbox includes the AlloyDB Control Logic Handler (CLH) and its dependencies.  
* **Centigrate:** A framework for building, deploying, and testing services in a sandbox environment. (Note: Centigrate is converging with ITS, see [go/centi-deprecate](https://goto.google.com/centi-deprecate)).  
* **Guitar:** ([go/guitar](https://goto.google.com/guitar)) A framework for running integration tests. LuSTI tests are often run via Guitar, especially because they interact with external systems (the sandbox) and can be long-running.  
* `lusti.Test`: The primary struct used within a LuSTI Go test. It wraps `*testing.T` and provides access to LuSTI framework functions, logging, and context.  
* `t.Lux()`: Accessor for AlloyDB (Lux) specific helper functions within the LuSTI framework.  
* `CreateCluster()`: A LuSTI function that sends API requests to the CLH in the sandbox to create an AlloyDB cluster, primary instance, and any read pools, as defined in the configuration.  
* **Subtests (`t.Run`)**: Used within the Go test to logically group and name individual operations (like a specific failover or resize attempt). This improves test output organization in Sponge.  
* **Implicit Assertions:** LuSTI operations like `Failover`, `SetFlagsTo`, etc., contain built-in checks. They don't just trigger the action; they also wait for it to complete and validate that the cluster is in a healthy state afterward. This reduces the need for explicit assertion code in the test body for common scenarios.  
* **Error Handling:** LuSTI methods typically fail the test immediately upon encountering an error (using `t.Fatal`), rather than returning errors to be handled by the caller. This simplifies test code.

**Execution Flow of the Codelab Test**

1. **Sandbox Startup:** You first start the AlloyDB sandbox environment using Centigrate aliases (`integration_e2e_env-setup`). This brings up the AlloyDB control plane and dependencies on Borg.  
2. **Test Invocation:** You run `blaze test ... //storage/testing/lusti/tests/random:lusti_lux_uniform_test_integration_e2e_env_in_a_cluster_no_env_deps ...`.  
   * The target name indicates it's running against an existing (`no_env_deps`) Centigrate environment running in a cluster (`integration_e2e_env_in_a_cluster`).  
   * `--existing_sandbox=alloydbadmin-satyasais-sandbox`: Tells the test which running sandbox to target.  
3. **Blaze Build:** Blaze builds the `random_test` Go binary.  
4. **Config Generation:** The `lusti_random_op_test` macro in the BUILD file, using helpers from `random.bzl`, constructs a large textproto string based on the arguments provided in the BUILD file. This textproto is passed as a command-line argument to the `random_test` binary.  
5. **Test Execution (Go):**  
   * The `random_test` binary starts.  
   * `lusti.NewConfigFromFlags` reads the textproto from the arguments.  
   * `lusti.Run` parses the textproto into the `confpb.Flags` message. Because `random_op_test` is set in the `oneof`, it extracts the `confpb.RandomOpTest` message.  
   * The callback function `func(t *lusti.Test, f *confpb.RandomOpTest)` is executed.  
   * **Cluster Setup:**  
     * `t.Lux().NewDBInstance()`: Converts the `LuxCluster` proto config into a Go struct.  
     * `t.Lux().CreateCluster()`: Makes API calls to the sandbox to create the AlloyDB cluster and instances.  
     * `cluster.Primary.LoadDataset()`: Populates data.  
   * **Random Operations:**  
     * The test loops `f.GetCount()` times.  
     * In each iteration, it selects an operation from `f.GetOps()` based on the defined weights.  
     * The selected operation (Failover, SetFlags, ReadPoolResize) is executed within a `t.Run()` subtest.  
     * The LuSTI functions called within the subtest interact with the sandbox's AlloyDB API, and perform their own internal validation.  
6. **Results:** Results of each subtest and the overall test are reported to Sponge, including logs and Dapper traces.

This configuration-driven approach, combined with powerful helper functions and macros, allows LuSTI to create complex integration tests with relatively little Go code.

