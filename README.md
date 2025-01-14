## Automated Build Pipeline with Jenkins Lab

Branched from https://github.com/bstiffler582/NEM_2024_SDT.git

LAB home: https://github.com/BA-Belgium/Lab-Jenkins-Build-And-Deploy-Lib
 
### Contents
1. [Introduction](#introduction)
2. [Jenkins Installation](#jenkins_install)
3. [Jenkins Initialization](#jenkins_init)
4. [Repository Setup](#repos)
5. [Jenkins Jobs](#jenkins_jobs)
6. [Automation Interface](#automation_interface)
7. [Testing](#testing)
8. [Outro](#outro)

<a id="introduction"></a>

### 1. Introduction

In the industrial automation industry, TwinCAT is uniquely suited for adopting modern software tooling and practices. Requests from customers looking to implement *continuous integration*, *continuous deployment* and *automated testing* with their PLC programs are becoming more and more frequent. The intent of this lab is to illustrate the overall landscape of CI/CD with TwinCAT, as well as to provide a hands-on demonstration of at least one path to realize these workflows with TwinCAT.

<a id="jenkins_install"></a>

### 2. Jenkins Installation

1. [Download and Install JDK 21](https://download.oracle.com/java/21/latest/jdk-21_windows-x64_bin.exe)
2. [Download and Install Jenkins LTS](https://www.jenkins.io/download/thank-you-downloading-windows-installer-stable/)
3. During install:
    - Point to JDK path @ `C:\Program Files\Java\jdk-21`
    - Select the "Run service as LocalSystem (not recommended)" option

<a id="jenkins_init"></a>

### 3. Jenkins Initialization

Jenkins typically runs as a remote server to act as the delegating service for a development team's build, test and deployment tasks. For this lab, we will keep things as simple as possible and run the Jenkins service right on our local machine.

1. Change service to run as user account
    - 🪟 + `r`, `services.msc`
    - Locate the **Jenkins** service, open properties
    - Log On tab > select "This account"
    - This fixes that "not recommended" step during installation ^^
2. Navigate to [localhost:8080](localhost:8080) in your favorite web browser
3. Copy/paste the initialization password at `C:\ProgramData\Jenkins\.jenkins\secrets\initialAdminPassword`
4. Don't create any more users (continue as `admin`)
5. Install suggested plugins
6. From Dashboard, select **Manage Jenkins**
    - Security -> Agents -> TCP Port for inbound... = "Random"

    ![Local Image](docs/pics/Agents%20-%20Inbound%20ports.png)
7. OPTIONAL for quick start: copy the configuration files of the TcAgent and Jobs described below in the appropriate folders of the Jenkins server
    ![Local Image](docs/pics/Jenkins%20Nodes.png)
    ![Local Image](docs/pics/Jenkins%20Jobs.png)

#### Jenkins Agent setup

The agent will be responsible for executing our build and test processes. With common software stacks, build agents are often small, transient containers or virtual machines that are quickly deployed as needed and then cleaned up. Since we are building for TwinCAT, we need an agent that has both the TwinCAT realtime and XAE (or Visual Studio).

1. "Set up an agent"
2. Give the new node a name, e.g. `TcAgent`
    - Select "Permanent Agent"
    ![Local Image](docs/pics/Agents%20-%20Create%20Agent.png)
    - Remote root directory = `C:\_jenkins\Agent`
    ![Local Image](docs/pics/Agents%20-%20Configure%20Agent.png)
    - All other node settings default
3. Open agent "Status" page and copy "Run agent from command line (Windows)" command. Something like:
```ps
curl.exe -sO http://localhost:8080/jnlpJars/agent.jar
java -jar agent.jar -url http://localhost:8080/ -secret 6b82f554bc9b0251f38ab30a7f2490d11b3b929dd6306d776f7bccadbd1c177c -name TcAgent -webSocket -workDir "c:\_jenkins\agent"
```
![Local Image](docs/pics/Agents%20-%20Commandline.png)

4. Open PowerShell **As Administrator** and run the command
Alternatively, put the command in a batch file and run it **As Administrator**  
We've now manually configured a custom build agent which is running locally and listening for jobs from the Jenkins server. Make sure we disable the "Built-in Node" agent that is pre-installed with the service, by putting the number of executors to 0.
![Local Image](docs/pics/Agents%20-%20Disable%20Default%20Agent.png)


> Imagine a large development team that pumps out several builds of different projects or micro-services daily. They likely need to be able to configure multiple *remote* agents, all with varying environments; different tech stacks, dependencies, build configurations, etc. Think about what the pipelines for the a huge development team might look like...

<a id="repos"></a>

### 4. Repository Setup

Before we create our Jenkins job, we need a way to trigger its execution. Typically, automated builds are triggered by a **push** or **merge** to a remote repository. For the lab we are going to create a couple repositories on our local machine. One will act as our 'local', and one as our 'remote'.
Feel free to try with a remote github repository, but be aware if you want to use guthub, you will need to install the github plugins.

> Remote repositories are usually just that: remote. They are hosted online somewhere like Github or Azure DevOps. There is no reason a remote repository can't be hosted right on our local filesystem, though.

Open a PowerShell window and make sure you can run the `git` command. If you get a "git is not recognized..." response, you likely just need to add a `PATH` environment variable. If you have not manually installed Git for Windows, it will still already be installed with Visual Studio in the following location:

```
C:\Program Files\Beckhoff\TcXaeShell\Common7\IDE\CommonExtensions\Microsoft\TeamFoundation\Team Explorer\Git\cmd
---
C:\Program Files\Microsoft Visual Studio\2022\Community\Common7\IDE\CommonExtensions\Microsoft\TeamFoundation\Team Explorer\Git\cmd
```
Now we will create two directories for our local and remote repositories. I will stick mine in a `repos` folder in the root for easy access:
```ps
cd C:\
mkdir repos
cd repos
```
Now, we will create an empty `remote` repository, and clone it to our `local` directory:
```ps
git init --bare remote
```
```ps
git clone remote local
```

Now we have an empty remote master branch (just like e.g. creating a new repository in Github), and we have a local clone as a working directory. Let's copy everything from the example repository in our local folder. Via the Git Changes window, we should see Visual Studio tracking our differences between local and remote. Go ahead and commit/push the addition of the TwinCAT project.

> Note: If you observe the contents of the remote folder, you will not see any of your project files, but I promise, they are there. This is how data is stored in remote Git repositories. Git is optimized for textual compression and keeps its own index of what data goes where. These files are sometimes called *blobs*. If you want to test it, feel free to clear the contents of the `local` folder and then re-clone `remote`.

Now that we have a valid remote repository with a project in it, let's create a **Job** in Jenkins.

<a id="jenkins_jobs"></a>

### 5. Jenkins Job

Create a new Job. Call it something like *TcBuildInstall Library*, and select a project type of "Freestyle Project". You can add a description if you want.

Settings:
1. Source Code Management 
    - Select **Git**
    - Point to the file path of our remote repository `C:\repos\remote`
    - Be aware if you want to use guthub, you will need to install the github plugins
2. Build Triggers
    - Select **Poll SCM**
    - Schedule value of `H/2 * * * *` ("cron job" format - once every 2 minutes)
3. Build Environment
    - Check "Delete workspace before build starts"
4. Build Steps
    - Select **Execute Windows batch command**
    - Enter command `echo job has run!`

Save / Apply the job.

With this configuration, the job will poll our `remote` repository for new commits every 5 minutes. If a change is detected, the latest changes are fetched to the workspace directory (`C:\ProgramData\Jenkins\.jenkins\workspace\[JobName]`) and the script is executed.

If we keep an eye on the Dashboard, sometime in the next two minutes (or so), we will see the server kick off the job. Click the job run and check out the results. Some useful pieces of information:
- The commit message, triggering event and diagnostics info (run time, etc.)
- The Console Output page with a full log of the job
    - (Hopefully) including our echo script
- The project files in our workspace directory

<a id="automation_interface"></a>

### 6. TwinCAT Automation Interface

All the standard pieces for an automated build pipeline are in place, so let's move on to the TwinCAT specific stuff. The most accessible means of automating a TwinCAT project build would be via the Automation Interface PowerShell API. The following script will open the solution (in the background), select the project and perform a build. 

 ```ps
$dte = new-object -com "TcXaeShell.DTE.17.0" # XaeShell64 COM ProgId
$dte.SuppressUI = $true # suppress VS interface

# open solution file
echo "Opening solution"
$slnPath = "$pwd\TwinCAT Project\TwinCAT Project.sln"
$sln = $dte.Solution
$sln.Open($slnPath)

echo "Building TwinCAT project"
$sln.SolutionBuild.Build($true)

echo "Exiting..."
$dte.Quit()
 ```

 For the lab a script named `tcBuild.ps1` is included in the root of the repository. Before committing the changes, modify the job in Jenkins to call the new build script. 
 
 Change:
 ```cmd
 echo job has run!
 ```
 to
 ```cmd
 powershell -ExecutionPolicy Bypass -File "tcBuild.ps1"
 ```

Commit, push, and wait patiently. If everything is sorted, we should see our job kick off and run the Automation Interface script to open and build the project. If the script was executed silently (`$SuppressUI = $true` in the script), we can verify the success of the job by checking the Console Output page, and the workspace directory for a build output of the library. For the lab the $SuppressUI flag is FALSE so everything will open up and happen in foreground so you can see what is happening. 

Also it is possible to look at the console output if the job kicks off.
![Local Image](docs/pics/Task%20-%20executing.png)
![Local Image](docs/pics/Task%20-%20Console%20Output.png)
![Local Image](docs/pics/Task%20-%20Console%20Output%20Result.png)

The final piece of the continuous integration pipeline would be to package up our build output for distribution. Navigate back to the job configuration page and scroll all the way down to "Post-build Actions". Add a new action of type **Archive the artifacts**, and enter the following in *Files to archive*:
```
FileIO.library
```
![Local Image](docs/pics/Task%20-%20Archiving%20Artifacts.png)

This is just telling the agent to grab the library and set it aside from the workspace directory. If we run our job again, we will now see build artifacts as part of the last successful job status. We can navigate these files and download archives for distribution.
![Local Image](docs/pics/Task%20-%20Archived%20Artifacts.png)

The next natural step would be *deployment* of these artifacts to some remote target. Jenkins is capable of this functionality with additional plugins, but that princess is in another castle (for today). It is not a leap to imagine using file transfer utilities and scripts to update remote target boot folders with the generated artifacts.

### 7. Outro

We have demonstrated automating the build and test process of a TwinCAT project using Jenkins, PowerShell and the Automation Interface API. With this exposure, hopefully you have gained familiarity with DevOps tooling and terminologies, as well as the continuous integration workflow. From the TwinCAT perspective, realizing this workflow in other comparable tools (e.g. Azure DevOps) would look very similar.
