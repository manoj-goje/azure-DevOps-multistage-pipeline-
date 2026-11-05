**Azure DevOps Java CI Pipeline**
Based strictly on your uploaded Step_1.pdf (setup steps and sample YAML). Where the file is silent, this guide says so instead of guessing.
0. Read this first: what your file proves and what it does not
What the file shows: a manually triggered CI pipeline for a Java/Maven project. It runs on a self-hosted agent (a VM you converted into an agent) and has five stages: compile, test, trivy_scan, code analysis, docker. The last stage builds a Docker image and pushes it to Azure Container Registry (ACR).
Gaps you must know before an interview (say these honestly, they show maturity):
1.	There is no deployment stage. The pipeline ends when the image is pushed to ACR. Nothing deploys to AKS, App Service, a VM or Container Apps. So the honest description is: CI plus image delivery to a registry. Do not claim deployment unless you really did it elsewhere.
2.	trigger: none means the pipeline runs only when someone clicks Run. No automatic trigger on commit.
3.	Stage names look missing. The extracted text shows - stage: followed directly by displayName. A valid YAML needs a name, for example - stage: compile. This is probably a PDF copy artifact (the text also glues lines like displayName: compilejobs:). Check your real file.
4.	The two Maven tasks use different subscription names: pay-as-you-go(1)(...) in compile and pay-as-you-go(...) in test. One is likely a typo or two different service connections. Both names must exist in Project settings.
5.	Trivy writes fs.html but uses --format table. The file is plain text with an .html name, it is never published as an artifact, and Trivy does not fail the build on findings (no --exit-code).
6.	No SonarQubePublish task and no quality gate check. Analysis is sent to SonarQube, but the pipeline does not wait for or enforce the result.
7.	The Dockerfile is not in the PDF. You cannot say what base image it uses. No image tag is set, so Docker@2 uses its default tag, which is the build ID ($(Build.BuildId)).
8.	Not present at all: variables:, condition:, dependsOn:, templates, environments, approvals, branch triggers, artifact publishing, the VM operating system, and the repo contents.
9.	Setup details not in the file: VM size and OS, how the agent was registered (PAT), ACR name and SKU, SonarQube server location.
1. Complete YAML Pipeline Explanation
1.1 The YAML, cleaned up (stage names added where the PDF dropped them)
trigger: none

pool:
  name: nitpoool
  demands:
    - agent.name -equals nittagent

stages:
- stage: compile
  displayName: compile
  jobs:
  - job: compile
    displayName: compile
    steps:
    - task: Maven@4
      inputs:
        azureSubscription: 'pay-as-you-go(1)(02ed2d1d-...)'
        mavenPomFile: 'pom.xml'
        goals: 'compile'
        publishJUnitResults: true
        testResultsFiles: '**/surefire-reports/TEST-*.xml'
        javaHomeOption: 'JDKVersion'
        mavenVersionOption: 'Default'
        mavenAuthenticateFeed: false
        effectivePomSkip: false
        sonarQubeRunAnalysis: false

- stage: test
  displayName: test
  jobs:
  - job: test
    steps:
    - task: Maven@4
      inputs:
        (same inputs, but goals: 'test')

- stage: trivy_scan
  jobs:
  - job: trivy
    steps:
    - task: CmdLine@2
      inputs:
        script: 'trivy fs --format table -o fs.html .'

- stage: code_analysis
  jobs:
  - job: sonarqube
    steps:
    - task: SonarQubePrepare@7
      inputs:
        SonarQube: 'sonar-conn'
        scannerMode: 'cli'
        cliScannerVersion: '7.1.0.4889'
        configMode: 'manual'
        cliProjectKey: 'sonarkey'
        cliProjectName: 'sonarkey'
        cliSources: '.'
        extraProperties: 'sonar.java.binaries=.'
    - task: SonarQubeAnalyze@7
      inputs:
        jdkversion: 'JAVA_HOME_17_X64'

- stage: docker
  jobs:
  - job: buildandpush
    steps:
    - task: CmdLine@2
      inputs:
        script: 'mvn package'
    - task: Docker@2
      inputs:
        containerRegistry: 'acr-conn0111'
        repository: 'classregistry'
        command: 'buildAndPush'
        Dockerfile: '**/Dockerfile'
1.2 The vocabulary (hierarchy)
Pipeline > Stage > Job > Step (task or script).
•	Pipeline: the whole YAML file.
•	Stage: a big phase (compile, test, scan...). Stages run one after another by default. A failed stage stops the later ones.
•	Job: a unit of work inside a stage. Each job runs on one agent. A stage can have several jobs that run in parallel.
•	Step: one action inside a job. A step is either a task (a packaged building block such as Maven@4) or a script (such as CmdLine@2).
•	@4, @7, @2 are task versions. Pinning the version keeps the pipeline from changing behaviour when Microsoft or the extension author releases a new major version.
Analogy: a car factory. The pipeline is the whole factory line. Stages are workshops (body, paint, safety check, packing). Jobs are the teams inside a workshop. Steps are individual actions by one worker. The agent is the worker who actually does the physical work. Azure DevOps is the manager who hands out instructions.
1.3 Line by line
trigger: none 
•	What: disables automatic runs (no CI trigger on push).
•	Why: a training project where you want control, or to avoid spending agent and ACR resources on every commit.
•	If removed: the default is to trigger on every push to any branch. If set to main, only main pushes would run it.
•	Interview line: 'It is manual-trigger for now. In production I would trigger on main and use PR validation through a branch policy.'
pool: / name: nitpoool
•	What: tells Azure DevOps which agent pool runs the jobs. nitpoool is a self-hosted pool (your own VM), not Microsoft-hosted (which would be vmImage: ubuntu-latest).
•	Why: your tools (Java, Maven, Git, Docker, Trivy) are installed on your VM, and you control the environment.
•	If removed: Azure DevOps uses a default pool. Your pipeline could land on a machine that lacks Trivy or Docker and fail.
•	Trade-off: a self-hosted agent must be patched and kept online by you. Hosted agents are clean and maintained by Microsoft, but you pay for parallel jobs beyond the free grant and tools may be missing.
demands: / - agent.name -equals nittagent
•	What: a rule saying 'only use an agent whose name equals nittagent'. Demands filter agents by capabilities (system and user-defined capabilities shown in the agent's Capabilities tab). agent.name is a built-in capability.
•	Why: pins the job to the one VM known to have all the tools.
•	If removed: any online agent in the pool could take the job. With one agent, the behaviour is the same. With several, a job could land on one missing tools.
•	If the agent is offline: the run waits in the queue ('No agent found in pool...') until it times out.
stages:
The list of phases. Each item begins with - stage:. Your file has no dependsOn, so each stage depends on the previous one automatically (sequential). That gives you the order compile > test > trivy_scan > code_analysis > docker.
Stage 1: compile (jobs > job: compile > steps > Maven@4)
•	displayName: the label shown in the pipeline UI. It is cosmetic.
•	task: Maven@4: the built-in Maven task. It runs mvn with your parameters.
•	azureSubscription: the name of an Azure Resource Manager service connection (Project settings > Service connections). In Maven@4 it is used mainly to authenticate to Azure Artifacts feeds with Entra ID. Because mavenAuthenticateFeed is false, it is not doing real work in this file. Note it, and do not over-explain it. If the named connection does not exist, the run fails at validation.
•	mavenPomFile: 'pom.xml': path to the Maven project file relative to the repo root. pom.xml lists dependencies, plugins and Java version. If it is missing or in a subfolder, Maven fails with 'no POM in this directory'.
•	goals: 'compile': the Maven goal. compile turns .java into .class files. It also downloads dependencies. It catches syntax and dependency errors early (fail fast).
•	publishJUnitResults: true and testResultsFiles: '**/surefire-reports/TEST-*.xml': after the run, collect JUnit XML reports (written by the Maven Surefire plugin) and show them in the pipeline's Tests tab. The ** means 'any folder depth'. In the compile stage no tests run, so there are no reports and the task may warn that none were found. This is harmless.
•	javaHomeOption: 'JDKVersion': choose the JDK by version number. Since no jdkVersion is given, the task uses its default (the agent's own JAVA_HOME). So the JDK on the agent decides the Java version. Java version is not stated in the file (only JAVA_HOME_17_X64 is mentioned in the Sonar step).
•	mavenVersionOption: 'Default': use the Maven already installed on the agent (the alternative is to point to a custom Maven path).
•	mavenAuthenticateFeed: false: do not inject Azure Artifacts credentials into Maven's settings. Dependencies come from public Maven Central as usual.
•	effectivePomSkip: false: an option used with feed authentication (it controls generating the 'effective POM' to find feed URLs). With authentication off, it has no practical effect.
•	sonarQubeRunAnalysis: false: do not run SonarQube through the Maven task. Sonar is run later in its own stage.
•	If the whole stage is removed: the test stage would still compile (Maven's test goal includes compile), so the pipeline still works. But you lose a separate, clearly labelled 'does it build' signal.
Stage 2: test
Same task, only goals: 'test'. Maven runs the lifecycle up to test: compile and then run unit tests (JUnit through Surefire). A failing test makes the step, job, stage and pipeline fail, and the later stages do not run. Reports are published to the Tests tab, so someone can see which tests failed without reading logs.
•	If removed: broken code could reach the Docker image. Note mvn package later also runs tests (unless -DskipTests), so tests would still run inside the docker stage, but late and without nice reporting.
Stage 3: trivy_scan (CmdLine@2)
•	CmdLine@2: runs a shell command (bash on Linux, cmd on Windows). The OS of your VM is not stated, but the tools suggest Linux.
•	script: 'trivy fs --format table -o fs.html .'
o	trivy: the open-source Aqua Security scanner (installed on your VM in Step 2).
o	fs: filesystem mode. It scans the project folder for vulnerable dependencies (read from files like pom.xml) and, by default, for hard-coded secrets.
o	--format table: human-readable table output.
o	-o fs.html: write the report to a file called fs.html. Because the format is table, this is plain text despite the extension. (To get a real HTML report you would use a Trivy HTML template.)
o	.: scan the current folder, which is the checked-out repo ($(Build.SourcesDirectory)).
•	Important limitations: no --exit-code 1 --severity HIGH,CRITICAL, so findings never fail the pipeline. The report is not published with PublishBuildArtifacts, so you must open it on the agent VM. Be ready to explain this and how you would fix it.
•	If removed: no security scanning of dependencies (DevSecOps / 'shift-left' security is lost).
•	If trivy is not installed on the agent: 'trivy: command not found' and the stage fails.
Stage 4: code_analysis (SonarQube)
This stage has two tasks that must stay together: Prepare configures, Analyze runs.
SonarQubePrepare@7 (from the SonarQube Marketplace extension, which must be installed in the Azure DevOps organization):
•	SonarQube: 'sonar-conn': the SonarQube service connection (server URL plus access token). Created in Project settings > Service connections.
•	scannerMode: 'cli': use the generic SonarScanner CLI (other modes are MSBuild for .NET and Maven/Gradle integration). This fits a Java project when you are not using the Maven scanner plugin.
•	cliScannerVersion: '7.1.0.4889': the exact scanner version to download. Pinning gives reproducible results.
•	configMode: 'manual': you write the settings in the YAML. The other mode reads a sonar-project.properties file.
•	cliProjectKey and cliProjectName: 'sonarkey': the unique ID and display name of the project in SonarQube.
•	cliSources: '.': analyze everything from the repo root.
•	extraProperties: 'sonar.java.binaries=.': extra Sonar settings. sonar.java.binaries tells Sonar where the compiled .class files are (needed for Java analysis). Here it points to . (the repo root), whereas many projects use target/classes. If Sonar complains about missing binaries, this line is the first thing to check.
SonarQubeAnalyze@7:
•	Runs the scan using the settings from Prepare and uploads results to the SonarQube server.
•	jdkversion: 'JAVA_HOME_17_X64': names an environment variable pointing to a JDK 17. The SonarQube server side needs Java 17 to run the scanner. It means a JDK 17 must exist on your agent and the variable must be defined there. (Microsoft-hosted images define this automatically. On your own VM you have to set it.)
Not present: SonarQubePublish@7, which waits for the quality gate result. Without it, a project with bugs and vulnerabilities still passes the stage.
•	If Prepare is removed: Analyze fails ('Prepare must run first').
•	If the stage is removed: no static code analysis (bugs, code smells, vulnerabilities, duplication, coverage if configured).
Stage 5: docker (job: buildandpush)
Step 1: CmdLine@2 with mvn package
•	Compiles, runs tests again, and packages the app into a .jar or .war in the target/ folder. The Dockerfile most likely copies that artifact into the image. That is an assumption, because the Dockerfile is not in the file.
•	If removed, docker build may fail with 'file not found' when it tries to copy the jar. If the Dockerfile builds from source itself (multi-stage build), this step would be unnecessary.
Step 2: Docker@2
•	containerRegistry: 'acr-conn0111': a Docker Registry service connection of type Azure Container Registry, pointing at the ACR you created in Step 4. It holds the login details so you never put passwords in YAML.
•	repository: 'classregistry': the image repository name inside the registry (image name). Registry name and repository name are different things: full image name is <acrname>.azurecr.io/classregistry:<tag>. Check whether 'classregistry' is your repository name or your ACR name.
•	command: 'buildAndPush': runs docker build and then docker push in one go. Other commands: build, push, login, logout.
•	Dockerfile: '**/Dockerfile': find the Dockerfile at any depth. The build context defaults to the Dockerfile's folder.
•	Tags are not set, so the default $(Build.BuildId) is used. Every run produces a unique, traceable image tag (for example classregistry:152). There is no latest tag unless you add tags:.
•	Needs on the agent: Docker installed, the agent's user in the docker group (or it fails with 'permission denied ... docker.sock'), and outbound network to *.azurecr.io.
•	If removed: you have no deployable artifact. The pipeline would only validate code.
1.4 Complete execution flow (what really happens)
1.	A person clicks Run pipeline (because trigger: none).
2.	Azure DevOps reads the YAML, validates it, and checks the service connections and permissions.
3.	It looks in pool nitpoool for an online agent named nittagent. If none is online, the job waits in the queue.
4.	The agent downloads the job, then checks out the repo automatically (an implicit checkout: self step) into its working folder.
5.	compile: Maven compiles the code.
6.	test: Maven runs unit tests, and the results appear in the Tests tab.
7.	trivy_scan: Trivy scans dependencies and writes fs.html on the agent.
8.	code_analysis: SonarQube Prepare then Analyze sends code quality results to the server.
9.	docker: mvn package creates the jar, then Docker builds the image and pushes it to ACR.
10.	The run is marked Succeeded (any failed stage stops everything after it).
What 'successful deployment' means here: a new image tagged with the build ID now sits in ACR. Deploying it is a separate step that is not in this YAML. A natural extension is a deploy stage that updates AKS, App Service or Container Apps, preferably with an Environment and manual approval.
1.5 Setup steps from your PDF (what they mean)
•	Step 1, import repository: copy the GitHub repo into Azure Repos (Repos > Files > Import). The pipeline then builds from Azure Repos.
•	Step 2, create VM and convert it into an agent; install Java, Maven, Git, Docker, Trivy: a self-hosted agent is a VM running Microsoft's agent software. It has to contain every tool the pipeline calls, because the pipeline just runs commands on that machine.
•	Step 3, make the agent online: the agent service must be running and able to reach dev.azure.com. 'Offline' agents cannot take jobs.
•	Step 4, create ACR: a private Docker registry in Azure to store images.
2. Graphical User Interface (GUI)
Honest note on screenshots: I cannot capture screenshots from your own Azure DevOps organization. Instead you get exact click paths (menu names as they appear in Azure DevOps) plus a description of what you should see. Take your own screenshots while you repeat these steps. In an interview, a screenshot of your own run is strong proof.
2.1 YAML vs GUI: who owns what
Item	Where it lives	Notes
trigger, pool, demands, stages, jobs, steps, task inputs	YAML (in the repo)	Versioned in Git, reviewable in pull requests
Agent pool nitpoool and agent nittagent (creation, registration, online status)	GUI + VM	The YAML only references them by name
Service connections: pay-as-you-go, acr-conn0111, sonar-conn	GUI (Project settings)	Hold credentials. YAML only uses the names, which must match exactly
SonarQube extension	GUI (Marketplace, Organization level)	Without it SonarQubePrepare@7 is an unknown task
ACR itself	Azure portal	The pipeline only pushes to it
Tools on the VM (Java, Maven, Git, Docker, Trivy)	The VM	Neither YAML nor GUI installs them
Pipeline permissions (who may use which pool or connection)	GUI	A first run often asks you to permit access
Secret variables, approvals, environments, retention	GUI	None are present in your file
Key idea: the YAML says what to do. The GUI and Azure hold where to run it (pool/agent) and what credentials to use (service connections).
2.2 Step-by-step setup in the GUI
A. Import the repository
1.	Open your project, then Repos > Files.
2.	Click the repo name at the top and choose Import repository.
3.	Source type Git, paste https://github.com/pawan071202/Java-Project-Azure-DevOps.git, name it, click Import.
4.	You should see the project files (pom.xml, src, probably a Dockerfile).
B. Create the agent pool
1.	Bottom-left Project settings > Pipelines > Agent pools (or Organization settings > Pipelines > Agent pools).
2.	Add pool > Pool type: Self-hosted, name nitpoool, tick Grant access permission to all pipelines, Create.
C. Turn the VM into an agent (nittagent)
1.	Create the Azure VM in the Azure portal (OS and size are not in your file).
2.	In Azure DevOps open the pool nitpoool > Agents > New agent. Download the package for your VM's OS.
3.	Create a Personal Access Token: top-right user icon > Personal access tokens > New token, scope Agent Pools (Read and manage).
4.	On the VM extract the package and run the agent configuration (config.sh on Linux). It asks for: organization URL (https://dev.azure.com/<org>), authentication type (PAT), the PAT, the pool name nitpoool, the agent name nittagent.
5.	Start it with run.sh, or install it as a service so it survives reboots.
6.	Back in the GUI, Agent pools > nitpoool > Agents: status must show Online (your Step 3). The Capabilities tab lists what the agent detected, including agent.name = nittagent, which is the value your demands rule matches.
D. Install and verify tools on the VM Check each one answers: java -version, mvn -version, git --version, docker --version, trivy --version. Also confirm the agent's user can run docker ps without sudo. Set JAVA_HOME_17_X64 if you rely on the Sonar step. If you install a tool after starting the agent, restart the agent so it re-detects capabilities.
E. Create ACR (Azure portal) Container registries > Create: subscription, resource group, registry name (globally unique, lower case), region, SKU (Basic is enough for learning). Your file does not state these values.
F. Create the service connections Project settings > Pipelines > Service connections > New service connection
1.	Azure Resource Manager for the subscription. Your YAML name looks like pay-as-you-go(...).
2.	Docker Registry > Azure Container Registry pointing at your ACR. Name it acr-conn0111 (the name the YAML uses).
3.	SonarQube (appears only after the extension is installed from the Marketplace): server URL plus token. Name it sonar-conn.
G. Create the pipeline
1.	Pipelines > Create Pipeline > Azure Repos Git > your repo > Starter pipeline (or Existing Azure Pipelines YAML file if the file is already committed).
2.	Paste the YAML. In the editor you can use Show assistant on the right: search for a task (Maven, Docker, SonarQubePrepare), fill the form, Add. The assistant writes the same inputs: you saw in section 1. That is the clearest demonstration of how GUI fields map to YAML keys.
3.	Save (commits the file to the repo) or Save and run.
H. Run it
1.	Pipelines > select your pipeline > Run pipeline, choose the branch, Run.
2.	On the first run a banner says the pipeline needs permission to use a resource (the pool or a service connection). Click View > Permit.
3.	Because of trigger: none, nothing starts on its own. You press Run each time.
I. Watch and verify
•	The run summary shows the stage graph: compile, test, trivy_scan, code_analysis, docker, each green, red or grey.
•	Click a stage, then a job, then a step to read its log.
•	Tests tab: unit test results from the published JUnit files.
•	Artifacts: nothing is published in your YAML, so it is empty. That is expected.
•	Azure portal > your ACR > Repositories > classregistry: the image and its tag (the build ID).
•	SonarQube dashboard: the project sonarkey with bugs, vulnerabilities, code smells.
•	Trivy report: the file fs.html sits in the agent's working folder on the VM (for a default Linux agent that is under _work/<n>/s).
2.3 Classic editor equivalent (useful for interview comparison)
YAML	Classic UI
pool	Agent pool dropdown on the Agent job
demands	Demands section of the Agent job
job	Agent job
task + inputs	A task added with the + button, and its form fields
stage	Not available in classic build pipelines. Classic release pipelines have stages
YAML is the recommended approach because the pipeline is code: versioned, reviewable, copyable between projects. The classic Releases area (Pipelines > Releases) is the older way to do CD. Your file covers only CI, so be ready to say how you would add CD.
2.4 Setting up your own screenshot set
Capture these seven, and you can walk an interviewer through the whole project: (1) imported repo, (2) agent pool with the agent Online, (3) service connections list, (4) the YAML in the editor, (5) a successful run's stage graph, (6) the Tests tab, (7) ACR repository with the image tag.
3. Architecture and Workflow Diagrams
3.1 Overall architecture, Git integration, agent pool and agent
 
architecture · Azure DevOps, one VM agent, two targets
How to read it: Azure Repos holds the code and the YAML. Azure Pipelines reads the YAML and queues each job to the agent pool nitpoool. Your VM, running the agent software as nittagent, polls Azure DevOps for work (outbound HTTPS, so the VM needs no inbound ports for this). It checks out the code from the repo, runs Maven, Trivy, the Sonar scanner and Docker locally, then sends results to SonarQube and pushes the image to ACR. The service connections supply the credentials those tasks use.
3.2 Pipeline execution workflow, build process and deployment flow
 
pipeline flow · 5 stages, 1 missing deploy step
Build and release in this project: the build (CI) part is complete: compile, test, scan, analyse, package into an image. The release (CD) part is missing from the YAML: the dashed Deploy box is where an AKS, App Service or Container Apps deployment would sit. In Azure DevOps terms, that would be a deploy stage targeting an Environment (with approvals), or a classic Release pipeline.
3.3 Connections between Azure DevOps and Azure resources
From	To	Connected by	Where in YAML
Azure Repos	Pipeline	Pipeline created from the repo (implicit checkout: self)	Not written, automatic
Pipeline	Agent pool and agent	Pool name and demand	pool.name, demands
Agent	Azure DevOps	PAT used once at registration, then outbound polling	Not in YAML
Pipeline	Azure subscription	Azure Resource Manager service connection	azureSubscription (Maven tasks)
Pipeline	SonarQube server	SonarQube service connection	SonarQube: 'sonar-conn'
Pipeline	ACR	Docker Registry service connection (type ACR)	containerRegistry: 'acr-conn0111'
Agent	ACR	docker push over the network, authenticated by that connection	Docker@2 buildAndPush
Rule to remember: the YAML refers to every external system by name. The secrets live in service connections, never in the fi
