# Java + Maven on EC2 — Beginner-Friendly DevOps Lab

This repository teaches the basics from the beginning: what Java and Maven do, how to install them on an Ubuntu EC2 server, how to build a small Java application, and how to open a simple web page using the EC2 public IP address.

You do not need to understand everything at once. Follow the sections in order and check each command's output.

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

```bash
cd /opt
wget https://archive.apache.org/dist/maven/maven-3/3.9.9/binaries/apache-maven-3.9.9-bin.tar.gz
```

### Step 3 — Extract Maven into `/opt`

`/opt` is a common place for manually installed optional software.

```bash
sudo tar -xzf apache-maven-3.9.9-bin.tar.gz -C /opt
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

### Step 3 — Allow port 8080 in the EC2 security group

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

### Step 4 — Open the page in your browser

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
2. The URL uses `http://`, the correct public IP, and `:8080`.
3. The security group has an inbound TCP `8080` rule from your current public IP.
4. The instance has a public IP and network route that permits access.
5. The Java application started without an error.

If your home/office public IP changes, the security-group **My IP** rule may need updating.

### Security reminder

This is an educational HTTP demo, not a production-secure web service. It uses plain HTTP (not HTTPS) and the demo server is intentionally simple. Restrict the inbound rule to your IP, stop the process when you finish, and remove the temporary rule if it is no longer needed. Never place passwords, tokens, or private information in this demo.

## 10. JAR, WAR, and EAR

| Package | Typical purpose | Simple explanation |
|---|---|---|
| **JAR** | Java library or application | A bundle of compiled Java classes and resources |
| **WAR** | Java web application | A web app package for a compatible servlet container such as Tomcat |
| **EAR** | Enterprise application in environments that support it | A bundle that can contain multiple enterprise modules |

This repository uses JAR packaging. A simple Java class does not become a Tomcat-ready WAR just by changing `<packaging>jar</packaging>` to `war`; a WAR needs the correct web-app structure and dependencies.

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
| `/opt/apache-maven-3.10.0/` | Maven program files, if installed there |
| `/home/ubuntu/maven-devops-learning/` | Your project files, if you cloned it there |
| `/home/ubuntu/.m2/repository/` | Downloaded dependencies and locally installed artifacts |
| `target/` inside the project | Generated class files and JARs |

The Maven installation folder and project folder are different. Run `mvn` from the project folder containing `pom.xml`.

## 15. Quick troubleshooting

- **`mvn: command not found`:** check Maven is installed and `PATH` in `~/.bashrc` is correct.
- **`The goal you specified requires a project to execute`:** you are probably not in the folder containing `pom.xml`. Run `pwd` and `ls -l pom.xml`.
- **`java: command not found` or `javac: command not found`:** install a JDK and check `java --version` and `javac --version`.
- **Browser cannot open port 8080:** check the Java process, public IP, inbound security-group rule, and network configuration.
- **Browser stops working after you close the terminal:** the Java process may have stopped. For this beginner lab, keep the terminal open while testing.

## 16. Broader DevOps picture

```text
Developer → Git commit → GitHub → Maven build and tests
           → JAR artifact → deployment/release process → server
```

Git/GitHub stores source code. Maven builds and packages it. A deployment process places/runs the artifact in its target environment. These are related but different jobs.

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
