# Java + Maven on AWS EC2: Build and Run a Java App (Beginner-Friendly DevOps Lab)

A step-by-step lab that shows how to install Java and Maven on an Ubuntu EC2 server, build a small Java application, and open a web page from it using the EC2 public IP address.

**Tech:** Java 21 (OpenJDK) · Maven · Ubuntu on AWS EC2 · Git/GitHub · AWS security groups

You do not need to understand everything at once. Follow the sections in order and check each command's output.

## What you will build and learn

By the end of this lab you will have:

- a running Ubuntu EC2 server you can reach over SSH,
- Java 21 and Maven installed and verified,
- a Maven project that compiles, tests, and packages successfully (`BUILD SUCCESS`),
- a small Java web server that you open in your browser at `http://<EC2-PUBLIC-IP>:8080`.

**Skills practiced:** Linux command line, SSH, package installation, environment variables, Maven build lifecycle, AWS security groups, basic network troubleshooting.

**Next step:** this lab is the foundation for the full CI/CD pipeline project (GitHub → Jenkins → Maven → SonarQube → Nexus → Tomcat): <https://github.com/Ramana175/Jenkins->

## Table of contents

1. [The big picture](#1-the-big-picture)
2. [What are Java, the JDK, and Maven?](#2-what-are-java-the-jdk-and-maven)
3. [Connect to an Ubuntu EC2 instance](#3-connect-to-an-ubuntu-ec2-instance)
4. [Install Java on Ubuntu EC2](#4-install-java-on-ubuntu-ec2)
5. [Install Maven on Ubuntu EC2](#5-install-maven-on-ubuntu-ec2)
6. [What is `pom.xml`?](#6-what-is-pomxml)
7. [Project layout](#7-project-layout)
8. [Build and run the simple standalone Java program](#8-build-and-run-the-simple-standalone-java-program)
9. [Run a Java web demo and open it through the EC2 public IP](#9-run-a-standalone-java-web-demo-and-open-it-through-the-ec2-public-ip)
10. [JAR, WAR, and EAR](#10-jar-war-and-ear)
11. [What is a test case?](#11-what-is-a-test-case)
12. [Dependencies and repositories](#12-dependencies-and-repositories)
13. [Maven lifecycles and common commands](#13-maven-lifecycles-and-common-commands)
14. [Where are Maven and the project stored?](#14-where-are-maven-and-the-project-stored)
15. [Quick troubleshooting](#15-quick-troubleshooting)
16. [Broader DevOps picture](#16-broader-devops-picture)
17. [Cleanup](#17-cleanup)

## Lab environment

| Item | Value used in this lab |
|---|---|
| Server | AWS EC2, Ubuntu |
| Java | OpenJDK 21 |
| Maven | 3.9.9 in the install steps (a newer version such as 3.10.0 also works) |
| Application port | `8080` (TCP) |
| SSH port | `22` (TCP) |

## 1. The big picture

Think of a Java application as a recipe.

- **Java code** is the recipe written by the developer.
- **JDK** gives the computer tools to compile and run Java code.
- **Maven** is the helper that builds the project, runs tests, and packages it.
- **EC2** is a computer rented from AWS that runs in the cloud.
- **A security group** is like a gatekeeper that decides which network connections are allowed to reach the EC2 computer.
- **The public IP address** is the internet-facing address people can use to reach that computer, if networking rules allow it.

```text
Developer writes Java code
          |
          v
       Maven reads pom.xml
          |
          +--> downloads libraries when needed
          +--> compiles Java code
          +--> runs tests
          +--> creates build output under target/
          |
          v
      Java application
          |
          v
EC2 instance + security-group rule for TCP 8080
          |
          v
Browser opens http://EC2-PUBLIC-IP:8080
```

**Important:** Maven builds the application. It is not itself a web server and it does not have a default web port. In this lab, the Java code listens on port `8080`, so we must allow that port through the EC2 security group.

## 2. What are Java, the JDK, and Maven?

- **Java** is a programming language used to write applications.
- **JDK (Java Development Kit)** contains the compiler (`javac`) and tools needed to build Java programs.
- **Maven** automates common project work: compiling, testing, managing libraries, and creating a package such as a JAR.
- **Git** tracks changes to code.
- **GitHub** hosts Git repositories online.

An easy way to remember it:

```text
Java = language
JDK = tools to compile and run Java
Maven = helper that builds the project
Git = tracks changes
GitHub = online place to store/share the code
```

## 3. Connect to an Ubuntu EC2 instance

You need an EC2 instance that is running and has a reachable public IPv4 address. In the AWS Console:

1. Open **EC2 → Instances**.
2. Select your instance.
3. Check that its state is **Running**.
4. Copy its **Public IPv4 address**.
5. Make sure you have the correct SSH key (`.pem`) and username.

For a standard Ubuntu EC2 image, the username is usually `ubuntu`.

**Note:** the public IP can change after you stop and start the instance, unless you attach an Elastic IP. Always copy the current IP from the console.

### From Windows PowerShell

Go to the folder where your downloaded `.pem` key is stored, then run this command, replacing the example values:

```powershell
ssh -i "my-ec2-key.pem" ubuntu@YOUR_EC2_PUBLIC_IP
```

Example shape only:

```powershell
ssh -i "C:\Users\YourName\Downloads\my-ec2-key.pem" ubuntu@203.0.113.10
```

`203.0.113.10` is a documentation example, not your real server address. Use the public IP shown for your instance.

### From Linux or macOS

```bash
chmod 400 my-ec2-key.pem
ssh -i my-ec2-key.pem ubuntu@YOUR_EC2_PUBLIC_IP
```

`chmod 400` makes the key readable only by you. SSH refuses keys that other users can read.

If SSH cannot connect, check that the instance security group allows **TCP 22** from your own public IP address, the instance has a public IP and reachable network route, and you selected the correct key and username. Do not open SSH to everyone (`0.0.0.0/0`) unless there is a specific, temporary reason and you understand the risk.

Official AWS guide: https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/connect-linux-inst-ssh.html

## 4. Install Java on Ubuntu EC2

Run these commands **after you have connected to EC2** and see a prompt similar to `ubuntu@ip-...`.

### Step 1 — Update the package list

```bash
sudo apt update
```

This refreshes Ubuntu's information about available software packages. It does not upgrade every installed package.

### Step 2 — Install the JDK

This repository targets Java 21, so install JDK 21 on Ubuntu:

```bash
sudo apt install openjdk-21-jdk -y
```

- `sudo` runs the command with administrator privileges.
- `apt install` installs software.
- `openjdk-21-jdk` is the Java 21 development kit.
- `-y` automatically answers yes to the installation confirmation.

### Step 3 — Verify Java and the compiler

```bash
java --version
javac --version
```

Both commands should show Java version 21. If Ubuntu's configured repositories do not provide that package, use an official supported JDK installation method for your Ubuntu release rather than downloading a random installer.

**Why check both?** `java` runs Java programs; `javac` compiles Java source code. Maven needs a JDK to compile the project.

Official Java/OpenJDK information: https://openjdk.org/

## 5. Install Maven on Ubuntu EC2

Maven is separate from Java, so installing Java does not automatically mean Maven is installed.

### Step 1 — Check whether Maven is already installed

```bash
mvn --version
```

If the command prints Maven and Java version details, Maven is already available. If you see `mvn: command not found`, continue below.

### Step 2 — Download the official Maven binary

Open the official download page and select a current stable binary archive: https://maven.apache.org/download.cgi

The following example shows the installation method for a specific archive name. If that version is no longer offered, use the current version and filename listed on the official download page.

Download into `/tmp`, because your normal user cannot write to `/opt`:

```bash
cd /tmp
wget https://archive.apache.org/dist/maven/maven-3/3.9.9/binaries/apache-maven-3.9.9-bin.tar.gz
```

### Step 3 — Extract Maven into `/opt`

`/opt` is a common place for manually installed optional software.

```bash
sudo tar -xzf /tmp/apache-maven-3.9.9-bin.tar.gz -C /opt
```

This creates `/opt/apache-maven-3.9.9`.

### Step 4 — Set environment variables for the Ubuntu user

Open the user's shell configuration file:

```bash
nano ~/.bashrc
```

`.bashrc` is a **file**, not a directory. Do not use `cd ~/.bashrc`.

Go to the end of the file and add these two lines:

```bash
export M2_HOME=/opt/apache-maven-3.9.9
export PATH="$M2_HOME/bin:$PATH"
```

Save in nano: press **Ctrl+O**, press **Enter**, then press **Ctrl+X** to exit.

Load the changes into the current terminal:

```bash
source ~/.bashrc
```

If you installed a different Maven version, make `M2_HOME` match the exact directory created under `/opt`.

#### Story: why we edit `.bashrc` and `PATH`

Imagine your Ubuntu server is a **school**, and you want to call a teacher named **mvn** (Maven).

- **Maven's folder (`/opt/apache-maven-3.9.9`) is the teacher's classroom.** Maven lives there, but the school does not know about it yet.
- **`M2_HOME` is the classroom address written on a card.** It says: "Maven lives here."
- **`PATH` is the school's list of places to look.** When you shout "mvn!", the computer checks each folder on this list, one by one. If the classroom is not on the list, it says `mvn: command not found`.
- **`export PATH="$M2_HOME/bin:$PATH"` adds the classroom to the list.** The `bin` folder is the door where the `mvn` program stands. The `:$PATH` part means "keep the old places too". Without it, you would erase the old list, and commands like `ls` would stop working.
- **`~/.bashrc` is your personal notebook.** Every time you open a new terminal, the computer reads this notebook first. So whatever you write here is remembered next time.
- **`source ~/.bashrc` means "read the notebook right now".** Without it, the new lines are only read the next time you open a terminal.

```text
You type: mvn --version
      |
      v
Computer checks the PATH list, folder by folder
      |
      +--> /opt/apache-maven-3.9.9/bin  -> finds "mvn" -> runs it
```

**Remember:**

1. `M2_HOME` is the address of Maven.
2. `PATH` is the list of places where commands are searched.
3. `.bashrc` saves the settings for every new terminal.
4. `source ~/.bashrc` applies them right now.
5. If the version number in the folder name is different, change it in `M2_HOME` too, or the address will be wrong.

### Step 5 — Verify Maven

```bash
mvn --version
which mvn
echo "$M2_HOME"
```

The output should show the Maven version and the installation path. If `mvn` is still not found, check the spelling and directory path in `~/.bashrc`, then run `source ~/.bashrc` again.

Official Maven installation guide: https://maven.apache.org/install.html

> If Maven is already installed manually at `/opt/apache-maven-3.10.0`, do not install a second copy just for this guide. Use that existing path and set `M2_HOME=/opt/apache-maven-3.10.0`.

## 6. What is `pom.xml`?

`pom.xml` means **Project Object Model**. It is the main configuration file Maven reads. It tells Maven the project's name and version, the Java version, libraries it needs, and build settings.

Run Maven commands from the project directory that contains `pom.xml`. If you run `mvn package` from `/opt/apache-maven-3.9.9/bin`, Maven will complain that there is no project there: that folder contains Maven's program, not your application.

## 7. Project layout

```text
maven-devops-learning/
├── README.md
├── .gitignore
├── pom.xml
└── src/
    ├── main/java/com/example/
    │   ├── App.java
    │   └── HelloHttpServer.java
    └── test/java/com/example/
        └── AppTest.java
```

- `src/main/java/` contains application code.
- `src/test/java/` contains automated tests.
- `pom.xml` contains Maven configuration.
- `target/` is generated by Maven and contains compiled classes and packages. It can be recreated by building again.
- `.gitignore` keeps generated files out of GitHub. It should contain at least:

```text
target/
```

## 8. Build and run the simple standalone Java program

First, enter the project folder (use the actual path where you cloned or copied it):

```bash
cd ~/maven-devops-learning
```

Check that you are in the right place:

```bash
pwd
ls -l pom.xml
```

Then run the build commands:

```bash
mvn validate
mvn compile
mvn test
mvn package
```

`BUILD SUCCESS` means that phase completed successfully. Maven normally downloads missing dependencies and plugins from configured repositories on the first build.

Run the console application:

```bash
java -cp target/classes com.example.App
```

Expected output:

```text
Hello from Maven!
```

This is a **standalone console application**: it runs in the terminal and prints a message. It does not listen on a network port, so a browser cannot connect to it using the EC2 IP address.

## 9. Run a standalone Java web demo and open it through the EC2 public IP

This next example is still a small standalone Java application, but it starts a tiny HTTP server so a browser can request a web page. It uses Java's built-in HTTP server; it does not require Tomcat for this learning demo.

### Step 1 — Build the code

From the folder containing `pom.xml`:

```bash
mvn clean package
```

### Step 2 — Start the HTTP server

```bash
java -cp target/classes com.example.HelloHttpServer
```

Expected terminal message:

```text
Server running on port 8080
```

Keep this terminal open. The Java process must keep running for the page to remain available. To stop it later, press **Ctrl+C**.

### Step 3 — Test it from inside the server first

Open a **second SSH session** to the same EC2 instance and run:

```bash
curl http://localhost:8080
sudo ss -lntp | grep 8080
```

`curl` should print `Hello from my EC2 Java server!`, and `ss` should show a Java process listening on port `8080`. If this works, the Java side is fine, and any remaining problem is networking (the security group or your IP).

### Step 4 — Allow port 8080 in the EC2 security group

A security group is a virtual firewall attached to the EC2 instance. It controls which inbound network connections can reach the instance and which outbound connections can leave it.

In the AWS Console:

1. Open **EC2 → Instances** and select your instance.
2. Open the **Security** tab.
3. Click the attached **security group**.
4. Open **Inbound rules** and choose **Edit inbound rules**.
5. Add a rule:
   - **Type:** Custom TCP
   - **Port range:** `8080`
   - **Source:** **My IP** for your own browser/computer, where possible
6. Save the rules.

Use `0.0.0.0/0` only if you intentionally want the page reachable from any IPv4 address, and only for a temporary demo. It exposes the port publicly. Do not open extra ports that the application does not need.

**Remember:** the security group rule only permits network traffic. It does not start the Java server. The Java program must also be running and listening on the port.

Official AWS security-group guide: https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/working-with-security-groups.html

### Step 5 — Open the page in your browser

In the EC2 Console, copy the instance's **Public IPv4 address**. In your own computer's browser, visit:

```text
http://YOUR_EC2_PUBLIC_IP:8080
```

Replace `YOUR_EC2_PUBLIC_IP` with the real address for your instance. Do not type the literal words `YOUR_EC2_PUBLIC_IP`.

You should see:

```text
Hello from my EC2 Java server!
```

How the connection works:

```text
Your browser
    |
    | HTTP request to public-IP:8080
    v
EC2 public network address
    |
    | Security group permits TCP 8080 from your IP
    v
Java HTTP server listening on port 8080
    |
    v
Browser receives "Hello from my EC2 Java server!"
```

If it does not work, check each point:
1. The Java server process is still running.
2. `curl http://localhost:8080` works on the server (Step 3).
3. The URL uses `http://`, the correct public IP, and `:8080`.
4. The security group has an inbound TCP `8080` rule from your current public IP.
5. The instance has a public IP and network route that permits access.
6. The Java application started without an error.

If a request **times out**, the security group or your IP is the usual cause. If it says **connection refused**, the Java server is not running on that port.

If your home/office public IP changes, the security-group **My IP** rule may need updating. Check your current IP at <https://checkip.amazonaws.com>.

### Keep the server running after you close the terminal (optional)

When you close the SSH session, the Java process stops. To keep it running in the background for a short test:

```bash
nohup java -cp target/classes com.example.HelloHttpServer > server.log 2>&1 &
```

Stop it later:

```bash
pkill -f HelloHttpServer
```

### Security reminder

This is an educational HTTP demo, not a production-secure web service. It uses plain HTTP (not HTTPS) and the demo server is intentionally simple. Restrict the inbound rule to your IP, stop the process when you finish, and remove the temporary rule if it is no longer needed. Never place passwords, tokens, or private information in this demo.

## 10. JAR, WAR, and EAR

| Package | Typical purpose | Simple explanation |
|---|---|---|
| **JAR** | Java library or application | A bundle of compiled Java classes and resources |
| **WAR** | Java web application | A web app package for a compatible servlet container such as Tomcat |
| **EAR** | Enterprise application in environments that support it | A bundle that can contain multiple enterprise modules |

This repository uses JAR packaging. A simple Java class does not become a Tomcat-ready WAR just by changing `<packaging>jar</packaging>` to `war`; a WAR needs the correct web-app structure and dependencies. The WAR and Tomcat deployment are covered in the CI/CD project linked at the top.

## 11. What is a test case?

A test case is code that checks whether another piece of code behaves as expected. `AppTest.java` checks the greeting returned by `App`.

JUnit is a Java testing framework. Other examples include Jest (JavaScript), pytest (Python), NUnit (C#), RSpec (Ruby), GoogleTest (C++), and Go's built-in `testing` package.

## 12. Dependencies and repositories

A **dependency** is a library a project needs. Maven normally checks its local repository first and downloads missing dependencies from configured remote repositories.

- **Local repository:** on this EC2 user's account, usually `/home/ubuntu/.m2/repository`.
- **Maven Central:** public repository for many Java libraries and Maven plugins.
- **Company repository:** tools such as Nexus Repository or JFrog Artifactory can host or proxy artifacts for an organization.

Check the local repository:

```bash
echo "$HOME"
ls -la ~/.m2
```

Maven Central: https://repo.maven.apache.org/maven2/

To customize the local repository, a user can create `~/.m2/settings.xml` with a `localRepository` setting. Only do this if you have a reason, such as moving the cache to a larger disk.

## 13. Maven lifecycles and common commands

Maven has three built-in lifecycles: `default`, `clean`, and `site`.

| Phase/command | Simple meaning |
|---|---|
| `mvn validate` | Check the project configuration |
| `mvn compile` | Compile application source code |
| `mvn test` | Run unit tests |
| `mvn package` | Create the configured package, a JAR in this project |
| `mvn verify` | Run checks to verify the package |
| `mvn install` | Save the artifact in the current user's local Maven repository |
| `mvn deploy` | Publish the artifact to a configured remote artifact repository |
| `mvn clean` | Remove generated output, usually `target/` |
| `mvn clean package` | Clean and then build/package |
| `mvn site` | Generate project documentation pages |

Maven runs earlier phases before a later phase in the same lifecycle. For example, `mvn package` compiles and tests before packaging.

**Important difference:** `mvn deploy` publishes a build artifact to a Maven repository. It does not deploy the running application to EC2 or Tomcat.

Skip running tests (use only when you have a reason):

```bash
mvn clean package -DskipTests
```

Skip test compilation and execution:

```bash
mvn clean package -Dmaven.test.skip=true
```

## 14. Where are Maven and the project stored?

In a common setup:

| Path | What is stored there? |
|---|---|
| `/opt/apache-maven-<version>/` | Maven program files (for example `3.9.9` or `3.10.0`) |
| `/home/ubuntu/maven-devops-learning/` | Your project files, if you cloned it there |
| `/home/ubuntu/.m2/repository/` | Downloaded dependencies and locally installed artifacts |
| `target/` inside the project | Generated class files and JARs |

The Maven installation folder and project folder are different. Run `mvn` from the project folder containing `pom.xml`.

## 15. Quick troubleshooting

| Problem | What to check |
|---|---|
| `mvn: command not found` | Check Maven is installed and `PATH` in `~/.bashrc` is correct, then run `source ~/.bashrc` |
| `The goal you specified requires a project to execute` | You are probably not in the folder containing `pom.xml`. Run `pwd` and `ls -l pom.xml` |
| `java: command not found` or `javac: command not found` | Install a JDK and check `java --version` and `javac --version` |
| `wget` fails with `Permission denied` | Download into `/tmp` (or use `sudo`), because a normal user cannot write to `/opt` |
| `UNPROTECTED PRIVATE KEY FILE` on SSH | Restrict the key permissions (`chmod 400` on Linux/macOS, or remove other users from the file on Windows) |
| `Permission denied (publickey)` on SSH | Wrong key or wrong username. Use the instance's key and `ubuntu` for Ubuntu images |
| `Address already in use` when starting the server | Another program is using port `8080`. Check with `sudo ss -lntp \| grep 8080` and stop it, or change the port |
| Browser times out on port 8080 | Check the inbound security-group rule and your current public IP |
| Browser says connection refused | The Java server is not running. Start it and test with `curl http://localhost:8080` |
| Page stops working after you close the terminal | The Java process stopped. Use `nohup` (see section 9) or keep the terminal open while testing |

## 16. Broader DevOps picture

```text
Developer → Git commit → GitHub → Maven build and tests
           → JAR artifact → deployment/release process → server
```

Git/GitHub stores source code. Maven builds and packages it. A deployment process places/runs the artifact in its target environment. These are related but different jobs.

## 17. Cleanup

When you finish the lab:

1. Stop the Java server (**Ctrl+C**, or `pkill -f HelloHttpServer`).
2. Remove the temporary inbound rule for port `8080` from the security group.
3. Stop or terminate the EC2 instance if you do not need it, so you are not charged for it. Also check for leftover EBS volumes and Elastic IPs.

## Official references

- Apache Maven: https://maven.apache.org/
- Maven downloads: https://maven.apache.org/download.cgi
- Maven installation: https://maven.apache.org/install.html
- Maven lifecycle: https://maven.apache.org/guides/introduction/introduction-to-the-lifecycle.html
- Maven settings: https://maven.apache.org/settings.html
- Maven Central: https://repo.maven.apache.org/maven2/
- JUnit 4: https://junit.org/junit4/
- AWS: Connect to a Linux EC2 instance: https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/connect-linux-inst-ssh.html
- AWS: Security groups: https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/working-with-security-groups.html
