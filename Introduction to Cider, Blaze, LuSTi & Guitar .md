### What is Blaze?

Think of Blaze as Google's main tool for building software from the source code stored in Google3 (Google's massive monorepo). It's like a very smart and efficient construction manager for code.

* **Purpose:** Blaze takes your source code (like .java, .cc, .py files) and turns it into executable programs, libraries, tests, or other outputs.  
* **Configuration:** You tell Blaze *how* to build your project using special files called BUILD files.  
* **Key Features:**  
  * **Dependency Management:** Blaze figures out all the pieces your code depends on and builds everything in the correct order.  
  * **Speed & Efficiency:** Blaze is fast because it can run many build steps in parallel using a distributed system called **Forge**. It also heavily caches build results, so it only rebuilds what's necessary.  
  * **Correctness:** Ensures builds are consistent and reproducible.  
  * **Scalability:** Handles the immense size and complexity of the Google3 codebase.

### Writing a BUILD File

Writing BUILD files is required to start a new project or add an existing one to Google3. Use the following template BUILD files to get started. Keep in mind while using these templates:

* A source file should usually only be in srcs for a single rule.  
  * To use a source file for more than one rule, make it the source to one library and then include the library as a deps to the other rules.  
  * The [py\_library and py\_binary](http://www.corp.google.com/eng/doc/python-developer-guide.html#BUILD) rules are notable exceptions to this guideline.  
* BUILD files must be formatted according to the [BUILD Style Guide](http://go/build-style).  
  * Use [Buildifier](http://go/buildifier) to check and fix BUILD file formatting.  
  * Run g4 fix in your [CLI](http://go/p4help#g4_fix) or [Cider](http://go/cider-user-guide/changelists.md#run-g4-fix) to trigger Buildifier on your behalf.

### Rules

A rule specifies the relationship between inputs and outputs, and the steps to build the outputs.

### 

#### Rule Parameters

Each rule accepts a set of parameters. Some common ones include:

* [name](https://g3doc.corp.google.com/devtools/blaze/g3doc/build-ref.html#name) \- A unique name for a target within the package. Other targets use this name to refer to the target.  
* [srcs](https://g3doc.corp.google.com/devtools/blaze/g3doc/build-ref.html#srcs) \- The list of source files processed to create the target including all checked-in code and any generated files required as inputs to this rule. Source files are [labels](https://g3doc.corp.google.com/devtools/blaze/g3doc/build-ref.html#labels)  
* [deps](https://g3doc.corp.google.com/devtools/blaze/g3doc/build-ref.html#deps) \- The list of directories and libraries the target depends on.  
* [data](https://g3doc.corp.google.com/devtools/blaze/g3doc/build-ref.html#data) \- The list of required data files read at runtime.

### 

### Files Blaze Uses to Perform Actions

Blaze primarily relies on the following types of files:

1. #### `1. BUILD` files: 

   * #### Define *what* to build (targets) and their dependencies. They use rules and macros.

   * **What they are:** These are the most crucial files for Blaze. The name is always `BUILD`.  
   * **Location:** You'll find a `BUILD` file in almost every directory containing source code within Google3. A directory with a `BUILD` file is called a "package".  
   * **Purpose:** They contain "rules" that tell Blaze:  
     * What source files (`.cc`, `.java`, etc.) are part of this package.  
     * What to build (these are called "targets"). Examples:  
       * `*_binary`: An executable program.  
       * `*_library`: A reusable piece of code (like a JAR or SO file).  
       * `*_test`: An automated test.  
     * What other targets this code depends on (the `deps`).  
   * **Language:** BUILD files are written in a specific format, using built-in functions and rules.  
   * **Example `BUILD` file snippet:**  
     `# This is a comment`  
     `# Defines a C++ library target named "my_lib"`  
     `cc_library(`  
         `name = "my_lib",`  
         `srcs = ["my_lib.cc"],  # Source files`  
         `hdrs = ["my_lib.h"],    # Header files`  
         `deps = [`  
             `"//base",  # Dependency on the base library`  
         `],`  
     `)`  
       
     `# Defines a C++ binary target named "my_app"`  
     `cc_binary(`  
         `name = "my_app",`  
         `srcs = ["my_app.cc"],`  
         `deps = [`  
             `":my_lib",  # Depends on my_lib in the same BUILD file`  
         `],`  
     `)`  
       
     `# Defines a C++ test target`  
     `cc_test(`  
         `name = "my_lib_test",`  
         `srcs = ["my_lib_test.cc"],`  
         `deps = [`  
             `":my_lib",`  
             `"//testing/base/public:gunit_main", # Dependency for testing`  
         `],`  
     `)`  
   * **Key parts of a rule:**  
     * `name`: A label to identify the target within the package (e.g., `"my_lib"`).  
     * `srcs`: A list of source files used to build this target.  
     * `deps`: A list of other targets this target needs to build or run.

2. #### 2\. Source Code Files:

   * These are your actual code files, like:  
     * `.cc`, `.h` (C++)  
     * `.java` (Java)  
     * `.py` (Python)  
     * `.go` (Go)  
     * `.proto` (Protocol Buffers)  
   * Blaze doesn't understand the *content* of these files directly, but the `BUILD` file rules tell it which compiler or tool to use for them.

3. #### 3\. .bzl files: 

   * #### Contain reusable logic (macros and custom rules) to avoid repetition in BUILD files. They help abstract complex build steps.

   * **What they are:** Files ending in `.bzl` contain custom build rules and macros written in a language called **Starlark** (which is a dialect of Python).  
   * **Purpose:** When the standard BUILD rules aren't enough, engineers write `.bzl` files to extend Blaze's capabilities, making BUILD files simpler and more powerful.  
   * **Usage:** These custom rules are brought into `BUILD` files using the `load()` function.

### 

### Basic Blaze Commands

To use Blaze, you typically run commands in your terminal from within a CitC client (your workspace in Google3). You can create or go to a workspace using g4d \<workspace\_name\>.

* **blaze build \<target\>:** Compiles the specified target. Targets are typically written as //path/to/package:target\_name.  
  * Example: blaze build //my/package:my\_app  
* **blaze test \<target\>:** Builds and runs the specified test target.  
  * Example: blaze test //my/package:my\_lib\_test  
* **blaze run \<target\>:** Builds the target and then runs it.  
  * Example: blaze run //my/package:my\_app

### When we Needing Proto, Binaries, and .bzl

This typically arises when your build process needs to **generate files based on structured data or perform transformations too complex for shell scripts.**

#### Analogy:

* **.proto:** The blueprint/schema for your data (like a database schema).  
* **Go/Python Binary:** A specialized machine tool (e.g., a CNC machine) that reads the blueprint and raw materials to produce a part.  
* **.bzl Macro:** The instructions for operating the machine tool, including feeding it the right blueprint and materials, and collecting the finished parts.  
* **BUILD File:** The order form specifying which blueprint and materials to use for today's job.

Protocol buffers are a mechanism for serializing structured data that is used extensively within Google. Outside of Google, similar data serialization formats include **JSON** and **XML**

### 

### Types of Execution Phases in Blaze:

1. #### 1\. Loading Phase: 

   * **Definition:** A phase for Declaration of the build.  
   * **What happens:** Blaze starts by reading and parsing the BUILD files relevant to the targets you asked it to build. This includes any .bzl files loaded via load() statements, which define custom rules and macros.  
   * **Goal:** To understand the targets, their declared source files, and their dependencies as written in the BUILD files.  
   * **Output:** An initial "**target graph**" representing the relationships between targets.   
   * **Errors:** Errors at this stage are usually syntax errors in BUILD or .bzl files, or problems like missing files referenced in load() statements.

   

2. #### 2\. Analysis Phase: 

   * **Definition:** A phase for planning of the build steps  
   * **What happens:** The Analysis Phase takes this initial graph and makes it more specific. A key thing Blaze does here is apply **configurations**. It makes a **Configured target graph**. Configurations are settings like:  
* What CPU architecture are we building for? (e.g., x86, Arm)  
* Are we building in "debug" or "optimized" mode (`-c opt`)?  
* Based on the Configured Target Graph, Blaze figures out the exact sequence of **actions** required to build everything. It makes an **action graph.**  
* Actions are the individual tasks like:  
  * "Compile file `a.cc` to produce `a.o`"  
  * "Link files `a.o` and `b.o` to produce executable `my_app`"  
  * **Errors:** Errors here often involve dependency issues (e.g., visibility problems, missing dependencies), type mismatches in rule attributes, or issues within the logic of custom Starlark rules.  
  * **Output:** Executable plan (**the Action Graph**), considering all the different contexts (**configured target graph**).  
  * **Optimization:** Blaze caches the analysis results. Incremental builds are fast if the BUILD files or dependencies haven't changed, as Blaze can reuse the existing action graph.  
    

3. #### 3\. Execution Phase: 

   * **Definition:** Actually run the commands from the Action Graph.  
   * **What happens:** This is where the actual building happens. Blaze executes the actions defined in the analysis phase.  
   * **Forge:** Most of these actions (like compiling code) are not run on your local machine. Instead, Blaze sends them to **Forge**, Google's distributed build and test system. Forge executes these actions in parallel across many machines, making builds much faster.  
   * **Caching:** Forge heavily caches action results. If the exact same action (same inputs, same command, same configuration) has been run before, Forge can return the cached output almost instantly.  
   * **Local Execution:** Some actions might run locally on your machine.  
   * **Goal:** To produce the final artifacts (binaries, libraries, test executables, etc.).  
   * **Errors:** Errors in this phase are typically compiler errors, linker errors, tool failures, or missing input files needed by an action. This phase usually takes the most time.

   

4. #### 4\. Testing Phase (for blaze test):

   * **What happens:** If you run blaze test, this phase follows the execution phase. Blaze runs the test targets that were built.  
   * **Environment:** Tests are also often run on Forge, in a controlled environment.  
   * **Goal:** To determine if the tests pass or fail.  
   * **Output:** Test results and logs.

### What is MPM package?

Imagine you've baked a cake (your software). You can't just hand someone the loose cake. You need to put it in a box to deliver it properly.

MPM (Midas Package Manager) is Google's system for "**boxing up**" software so it can be deployed to run on production servers (mostly on Borg).

Here's the breakdown:

1. What is an MPM Package?  
   * It's a bundle containing all the files your application needs to run. This typically includes:  
     * The executable binaries (the main program).  
     * Configuration files.  
     * Data files.  
   * Think of it like a super-secure, versioned ZIP file, but designed for Google's production environment.  
   * Each time you build a package, it gets a unique Version ID.  
2. How do you tell Blaze to create an MPM package?  
   * You add a special rule called genmpm to your BUILD file.  
   * This genmpm rule specifies:  
     * package\_name: A unique name for your package (e.g., my/team/my\_server).  
     * srcs: The Blaze targets (like cc\_binary, java\_binary, data files, etc.) that should be included in the box.

\# Example BUILD file snippet

\# This builds the actual program

cc\_binary(

    name \= "my\_server\_binary",

    srcs \= \["my\_server.cc"\],

    \# ... other dependencies

)

\# This rule tells Blaze how to package it into an MPM

genmpm(

    name \= "my\_server\_mpm",

    package\_name \= "my/team/my\_server",  \# The unique name in the MPM system

    srcs \= \[

        ":my\_server\_binary",  \# Include the binary

        \# You could add config files here too

    \],

)

3. How does Blaze build the MPM?  
   * When Rapid runs its "Create Candidate" workflow, it usually tells Blaze (via a tool called BuildRabbit) to build the target specified in the genmpm rule (e.g., //my/team/project:my\_server\_mpm).  
   * Blaze first builds all the srcs listed in the genmpm rule (like :my\_server\_binary).  
   * Then, Blaze bundles these outputs together, creates the package, and uploads it to the Midas package storage system. This newly created package version gets a unique ID.  
4. How does Rapid use the MPM package for deployment?  
   * Building: As above, Rapid uses Blaze to build the genmpm target and create the package version. Rapid automatically adds labels to this new version, including the name of the release candidate (e.g., my\_project\_20250821\_RC00).  
   * Deploying: Rapid deployment workflows don't usually copy the whole package around. Instead, they typically do things like:  
     * Applying Labels: Attaching a specific label (e.g., live, staging, prod) to the MPM package version created earlier.  
     * Updating Borg Jobs: Borg (Google's cluster management system) is configured to run a specific MPM package, often identified by a label. When Rapid moves the live label from an old version to the new version, it signals to Borg (often via Annealing) to update the running jobs to use the files from the new MPM package version.

#### Analogy:

* Your Code: The recipe and ingredients for a cake.  
* genmpm rule in BUILD: Instructions for how to box up the cake.  
* Blaze: The baker who bakes the cake AND puts it in the box.  
* MPM Package Version: The individually boxed cake, with a unique order number (Version ID). Stored in the Midas warehouse.  
* Rapid: The delivery service.  
  * It orders the cake to be baked and boxed (runs blaze mpm).  
  * To deploy, it changes the label on the box in the warehouse (e.g., "This box is now 'live' for delivery").  
  * Borg: The customer, who always picks up the box marked 'live' from the warehouse.

## Introduction to Cider, LuSTi & Guitar 

### Goals:

* Understand the development workflow using Cider.  
* Learn the structure and components of a LuSTI test.  
* Know how to execute LuSTI tests using Guitar.

### Workflow using Cider:

#### **1\. Connecting CloudTop to Cider:**

1. **Access Cider V:** Open Chrome and navigate to `cider/`. 

2. **Install Required Chrome Extensions:** To allow Cider V to communicate with your Cloudtop, you need the following extensions installed in Chrome:  
   1. **Cider Connector:** Facilitates the secure connection between Cider V and your Cloudtop terminal. See [go/cider-v-terminal](https://goto.google.com/cider-v-terminal).  
   2. **Google Security Key Extension (SKE):** Manages your Corp SSH certificates. See [go/new-ske](https://goto.google.com/new-ske).	  
3. **Install `cidermux` on your Cloudtop (Recommended):** `cidermux` enhances the terminal experience by providing connection persistence across Cider V reloads or network changes. Open a terminal directly on your Cloudtop (e.g., via SSH or CRD) and run:

   `sudo apt install cidermux`

4. Enable `cidermux` in Cider V settings (Ctrl+, or Cmd+,), search for `cidermux`, and check the box.

5. **Connect to Cloudtop Terminal within Cider V:**  
   1. Open Cider V.  
   2. Open the integrated terminal:  
      1. Press `Ctrl+Shift+` \`.  
      2. Or, open the Command Palette (`Ctrl+Shift+P` or `Cmd+Shift+P`) and type "Terminal: Create New Terminal".  
   3. When prompted for the SSH host, enter your Cloudtop's hostname (e.g., `your-cloudtop-name.c.googlers.com`). You can find this at [go/mytech](https://goto.google.com/mytech).  
   4. Touch your security key when prompted to authenticate.

#### **2\. Cider Workflow with g4/Piper:**

		

1. **What is Piper?** ([go/piper](https://goto.google.com/piper)): Google's main in-house source code management system, similar to Perforce.  
2. **What is Cider V?** ([go/cider-v](https://goto.google.com/cider-v)): Google's web-based IDE, tightly integrated with Piper and other Google tools.  
3. **Client in the Cloud (CitC):** ([go/citc](https://goto.google.com/citc))  
   1. Your personal view of the source code repository. Changes you make are isolated to your CitC client until you submit them.  
4. **Basic Cider-Piper Workflow:**  
   1. **Create a Workspace:**  
      1. In Cider: `Workspace > New Piper Workspace`. Give it a name (e.g., `nova_testing`).  
      2. **Command Line:**

         `g4d -f nova_testing`

      3. This creates the workspace if it doesn't exist and navigates into its `google3` directory.  
   2. **Edit Files:** Make code changes directly in the Cider editor.  
   3. **Open files for Edit/Add:** *\<No need to perform this step unless cider fails to add\>*  
      1. Cider often handles this **automatically**. New files are usually marked as "untracked" and existing files as "edited".  
      2. Files need to be part of a Changelist (CL). Cider typically prompts you to create one or add to the `default` CL.  
      3. **Command Line Equivalents:**

         `g4 add path/to/your/new_file.go  # To add a new file`

         `g4 edit path/to/your/existing_file.go # To edit an existing file`

   4. **Create/Modify a Changelist (CL):**  
      1. In Cider: The "Source Control" tab will show changes. You can create a new CL or modify an existing one, providing a description, reviewers, bugs, etc.  
      2. **Command Line:**

         `g4 change --desc “<add description for cl>”`

      3. This will create a **CL number**. You can view that CL, via Critique [cl/793469587](http://cl/793469587).  
   5. **Upload Snapshot to Critique:**  
      1. In Cider: There's usually a button to "Upload Snapshot" or it happens automatically when you request a review.  
      2. **Command Line:**

         `g4 upload`

      3. This makes your changes visible in Critique for review.  
   6. **Syncing Your Workspace:** To get the latest code from Piper's head:  
      1. In Cider: `Workspace > Sync to Head`.  
      2. **Command Line:**

         `g4 sync`

   7. **Code Review:** Interact with comments in Critique. Make changes in Cider, and they will be automatically saved in your CitC client. Re-upload snapshots to Critique as needed.  
   8. **Submitting:** Once approved, submit the CL through Critique.  
      1. **Command Line:**

         `g4 submit -c <cl_number>`

5. **Useful g4/p4 Command Line Commands:** ([go/g4-cheat](https://goto.google.com/g4-cheat))  
   1. `g4d <client_name>`: Navigate to an existing client.  
   2. `g4d -f <client_name>`: Create and navigate to a client.  
   3. `g4d <CL_number>`: Navigate to the client associated with a specific CL.  
   4. `g4 p` or `g4 pending`: List pending CLs and files in the current client.  
   5. `g4 status`: Show files opened in the current client.  
   6. `g4 sync`: Sync client to the latest version in Piper.  
   7. `g4 add/edit/delete <file>`: Stage file changes.  
   8. `g4 revert <file>`: Revert changes to a file.  
   9. `g4 mail -m <reviewer>`: Request a code review.  
   10. `g4 submit`: Submit the current CL (if it's ready).  
   11. `g4 nothave`: List files in your directory that are not in the repository.

### LuSTI Tests: Writing and Structure

* **What is LuSTI?** ([go/lusti](https://goto.google.com/lusti))  
  * (Lux \+ Speckle Testing Infrastructure) A framework to simplify writing integration tests for AlloyDB (Lux) and AlloyDB Omni (Smurf/Nova).  
  * Provides abstractions for instance creation, configuration, and operations.

  **Key Files in a LuSTI Test (using [cl/793469587](https://cl.corp.google.com/793469587) as example):**

* #### **`1. config.proto`**: ([`//storage/testing/lusti/src/config.proto`](https://source.corp.google.com/piper///depot/google3/storage/testing/lusti/src/config.proto))

1. Defines the configuration structure for tests.  
   1. **Purpose:** This file defines the structure of configuration data that your test can receive. Think of it as a blueprint for the settings your test needs.  
   2. **Key Message:** `Flags`. This is the root message for any LuSTI test configuration. It's passed the test as a textproto.  
   3. **Structure:**  
      1. `SharedFlags shared`: Contains settings common to *all* LuSTI tests (e.g., project, environment).  
      2. `oneof TestCaseFlags`: Contains messages specific to *each type* of test. Your `NovaTest` is added here. This design ensures a test only includes configuration relevant to it.  
* You added `NovaTest` message to hold settings specific to your Nova tests, like `SmurfInstance` details.

  message NovaTest {

    SmurfInstance smurf\_instance \= 1;

  }

  // This is added to the oneof TestCaseFlags in the Flags message

  // oneof TestCaseFlags {

  //   ...

  //   NovaTest nova\_test\_flags \= 76;

  // }


* #### **`2. .go` Test File**: ([`//storage/testing/lusti/tests/omni_nova/omni_nova_test.go`](https://source.corp.google.com/piper///depot/google3/storage/testing/lusti/tests/omni_nova/omni_nova_test.go))

1. The core test logic.  
   1. `lusti.Run()`: Entry point, handles setup and teardown. It passes the test-specific flags (e.g., `*confpb.NovaTest`).  
   2. `lt.Smurf()`: Accesses Smurf/Nova specific helpers.  
   3. `NewDBInstance()`: Prepares instance parameters.  
   4. `CreateInstance()`: Creates the actual Nova VM instance.  
   5. `smurf.RunRemoteTestOnClient()`: Executes commands/checks on the created VM.  
      package random\_test // package \<name\>\_test  
        
      import (  
      	"context"  
      	"testing"  
      	// ... other imports  
      	confpb "google3/storage/testing/lusti/src/config\_go\_proto"  
      	"google3/storage/testing/lusti/src/lusti"  
      	"google3/storage/testing/lusti/src/smurf"  
      )  
        
      func TestInstallAndDeploy(t \*testing.T) {  
      	lt := \&lusti.Test{  
      		T:            t,  
      		ConfigSource: lusti.NewConfigFromFlags,  
      	}  
      	lusti.Run(context.Background(), lt, func(t \*lusti.Test, f \*confpb.NovaTest) {  
      		// ... (rest of the test logic as in the CL) ...  
      		instanceCfg := f.GetSmurfInstance()  
      		if instanceCfg \== nil {  
      			t.Fatalf("Instance configuration not found in test flags")  
      		}  
      		// ... create instance and run tests ...  
      		lt.Infof("Successfully created Smurf instance")  
      	})  
      }

   

* #### **`3. .bzl` Starlark Macro File**: ([`//storage/testing/lusti/tests/omni_nova/omni_nova.bzl`](https://source.corp.google.com/piper///depot/google3/storage/testing/lusti/tests/omni_nova/omni_nova.bzl)) 

1. Defines custom macros functions to generate BUILD targets with specific configurations.  
2. **Mechanism:** These macros primarily perform string formatting to build the final textproto string that will be passed as an argument to the Go test.  
3. This file defines a reusable function (`alloydbomni_nova_test`) to generate multiple, similar test targets in the `BUILD` file.  
   1. `alloydbomni_nova_test` macro:  
      1. Takes a `base_test` (the `lusti_multi_env_go_test` target).  
      2. Constructs a `textproto` string conforming to `config.proto`. This is where you inject the `nova_test_flags` including `smurf_instance` details.  
      3. Calls `lusti_multi_env_variant` to create a new, configured test target. create the actual test targets.

   load("//storage/testing/lusti/src:lusti.bzl", "TYPE\_SMURF", …)

      

      def alloydbomni\_nova\_test(name, base\_test, \*\*kwargs):

          \# ... (as defined in the CL) ...

          textproto \= """

             shared: {{ ... }}

             nova\_test\_flags: {{

                smurf\_instance: {{ {smurf\_instance} }}

              }}

          """.format(...)

          lusti\_multi\_env\_variant(

              name \= name,

              \# ...

              textproto \= textproto,

              tags \= \[TYPE\_SMURF\],

              \*\*kwargs

          )

      

* #### **4\. BUILD File for Tests**: ([//storage/testing/lusti/tests/omni\_nova/BUILD](https://source.corp.google.com/piper///depot/google3/storage/testing/lusti/tests/omni_nova/BUILD))

**Purpose:** This file tells Blaze (Google's build system) how to build and run your tests.

1. lusti\_multi\_env\_go\_test: Defines the *base* Go test binary without specific LuSTI runtime flags.  
2. alloydbomni\_nova\_test: Uses the macro from the .bzl file to create the *runnable* test target, injecting the configuration.

   \# Base test, no specific nova config

   lusti\_multi\_env\_go\_test(

       name \= "omni\_nova\_test",

       srcs \= \["omni\_nova\_test.go"\],

       \# ... deps ...

   )

   

   \# Runnable test variant with config injected by the macro

   alloydbomni\_nova\_test(

       name \= "alloydbomni\_nova\_test",

       base\_test \= ":omni\_nova\_test",

       \# ... other params

   )

   

**How they connect:** The `.bzl` macro is key. It takes the generic Go test and uses `lusti_multi_env_variant` to bundle it with a specific `textproto` configuration. The Go test then parses this proto at runtime.

##### Architecture Diagram: [https\://graphviz.corp.google.com/\#a394186d8b7830c5d9bb994600358692](https://screenshot.googleplex.com/6AvsE9gcr2EYKgE.png)

##### **Key Terms and Concepts**

* **Textproto:** A human-readable text format representing a Protocol Buffer message. LuSTI uses this to pass the extensive configuration to the test binary.  
* **Starlark (BZL):** The configuration language used by Blaze and in BUILD files. Macros in `.bzl` files are functions that generate BUILD rules.  
* **Macro:** A function in a `.bzl` file that can be called from a BUILD file to generate rules. `lusti_random_op_test` is a macro.  
* **Sandbox:** An isolated environment to run tests without affecting production. For AlloyDB, this is typically managed by **Centigrate** ([go/centigrate](https://goto.google.com/centigrate)). The sandbox includes the AlloyDB Control Logic Handler (CLH) and its dependencies.  
* **Centigrate:** A framework for building, deploying, and testing services in a sandbox environment. (Note: Centigrate is converging with ITS, see [go/centi-deprecate](https://goto.google.com/centi-deprecate)).  
* **Guitar:** ([go/guitar](https://goto.google.com/guitar)) A framework for running integration tests. LuSTI tests are often run via Guitar, especially because they interact with external systems (the sandbox) and can be long-running.

### 

### 

### Guitar Tests: Executing LuSTI Tests

* **What is Guitar?** ([go/guitar](https://goto.google.com/guitar))  
  * Google's standard framework for running integration tests.  
  * Workflows are defined in BUILD files.  
* **Guitar BUILD File**: ([`//storage/alloydb/admin/testing/guitar/omni_nova/BUILD`](https://source.corp.google.com/piper///depot/google3/storage/alloydb/admin/testing/guitar/omni_nova/BUILD))  
  * `guitar_workflow_test`: Defines a workflow.  
  * `guitar.Tests`: Specifies which tests to run.  
  * **`targets`**: Crucially, this MUST point to the target generated by your `.bzl` macro (which has the config), NOT the base `lusti_multi_env_go_test`.


  load("//testing/integration/guitar/build\_defs:guitar\_workflow.bzl", "guitar", "guitar\_workflow\_test")


  guitar\_workflow\_test(

      name \= "omni\_nova\_test\_workflow",

      integration\_test \= guitar.IntegrationTest(

          tests \= \[

              guitar.Tests(

                  execution\_method \= "LOCAL",

                  targets \= \[

                      \# This is the target from omni\_nova.bzl

                      "//storage/testing/lusti/tests/omni\_nova:alloydbomni\_nova\_test",

                  \],

              ),

          \],

      ),

  )

* **Running the Workflow:**  
  `# Make sure you are in your workspace`  
  `# g4d nova_testing`  
    
  `# Run the workflow from your local CitC client changes`  
  `guitar run -w //storage/alloydb/admin/testing/guitar/omni_nova:omni_nova_test_workflow --version=citc:LOCAL --cluster=alloydb-manual`  
  * `-w`: Workflow target.  
  * `--version=citc:LOCAL`: Use code from your current CitC workspace.  
  * `--cluster`: Which Guitar cluster to use for execution.  
* **Viewing Results:** Results are streamed to Fusion (a TestFusion link is provided by the command).

**Alloydb:** Enterprise grade postgres sql database for high performance. This is a fully managed google cloud service.

**Alloydbomni:** This is an on-premise version of alloydb, managed by customers on their infrastructure.

**Replication:** The process of continuously updating read-only db’s from the primary db to ensure HA.  
It supports async and sync replication.  
**Async:** The primary database commits a transaction and proceeds to the next one without waiting for confirmation from the replica  
**Sync:** ensures high data integrity by requiring a transaction to be fully written to both the primary and the  replica, it will wait for an acknowledgement to the client.   
**Backup:** For backups and open source components to be placed.

**Alloydbomni monitor:** This rpm can integrate with alloydbomni and will provide monitoring metrics of alloydbomni which customer can track, He will integrate this endpoint to monitoring tools such as grafana.

**Patroni:** It is an open source tool used to automate HA, it is responsible for automatic failovers, leader election and replication management. 

**Resilient HA** mainly focus on high availability and data integrity, where **scalable HA** focuses on performance and capacity, where it allows database to handle many request.

**ETCD:** It is a Distributed configuration store, which acts as source of truth, it stores cluster state 

**Keepalived:** It provide high availability by managing a floating Virtual IP, ensures the database traffic moves to a standby node if the primary node fails.

**HAProxy:** Is used as network traffic load balancer by utilizing vrrp in layer 7   
HAProxy balances connections to multiple backend AlloyDB instances, while Keepalived provides a single virtual IP (VIP) that moves between nodes to prevent downtime

**Pgbouncer:** It is a Connection pooler for PostgreSQL, it acts as middleman between your application and the AlloyDB Omni database, efficiently managing active connections to optimize performance. (Port:6432)  
**Backup:**

* **Pgbackrest:** The tool which is responsible for physical backup and restore for alloydb omni to local or cloud storage, It can perform Full, differential, incremental backups also including PITR   
  * **Full:** Complete backup  
  * **Differential:** Only the changes which aren’t backup from the last full backup (delta backup), Skips unchanged files.  
  * **Incremental:** It copies only the files that have changed since the last successful backup.

  **PITR:** PITR enables restoring the AlloyDB Omni database to a specific, precise moment in time.


* **What is LuSTI?** ([go/lusti](https://goto.google.com/lusti))  
  * (Lux \+ Speckle Testing Infrastructure) A framework to simplify writing integration tests for AlloyDB (Lux) and AlloyDB Omni (Smurf/Nova).  
  * Provides abstractions for instance creation, configuration, and operations.  
*   
* **Guitar:** ([go/guitar](https://goto.google.com/guitar)) A framework for running integration tests. LuSTI tests are often run via Guitar, especially because they interact with external systems (the sandbox) and can be long-running.

A typical integration test looks like: [cl/869544062](http://cl/869544062) 

**Ansible:** 

Ansible is an automation tool used for configuration management, application deployment, and task automation. 

* It operates via ssh with a simple and secure connection from the control node to our db nodes.