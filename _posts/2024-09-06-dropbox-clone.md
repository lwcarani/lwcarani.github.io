---
title: Building My Own Dropbox
author: luke
date: 2024-09-06 12:00:00 +0500
categories: [Software Engineering, Coding Projects]
tags: [programming, coding projects, java, aws, cognito, s3]
---

In this blog post, I'm going to cover another [John Crickett](https://github.com/JohnCrickett) [coding challenge](https://codingchallenges.fyi/challenges/challenge-dropbox/) that I just completed - building my own Dropbox Clone. Dropbox is a cloud-based file hosting service that allows you to synchronize your files between different people and different devices. 

The full code for this project can be found at my Github [here](https://github.com/lwcarani/dropbox-clone).

## About
This project is a command-line interface (CLI) application (written in Java) that mimics some of the core functionalities of Dropbox, allowing users to synchronize files between their local machine and cloud storage (in this particular implementation, Amazon Web Services (AWS) Simple Storage Service (S3)). I've been meaning to brush up on Java, and learn how to use the Spring framework, so this seemed like the perfect project. I've also used Google Cloud and Firebase before, but not AWS, so I chose to AWS as the backend to learn more about the APIs and building on AWS services. For this project, I only needed AWS Cognito for user authentication, and AWS S3 for file storage and retrieval.

## Technology Stack

- Java 17
- Spring Boot 3.1.0
- AWS SDK for Java 1.12.465
- AWS S3 for file storage
- AWS Cognito for user authentication

## Setup

1. Ensure you have Java 17 installed.
2. Clone the repository:
   ```
   git clone https://github.com/lwcarani/dropbox-clone.git
   ```
3. Navigate to the project directory:
   ```
   cd dropbox-clone
   ```
4. Create an `application.properties` file in the `src/main/resources` directory with the following content:
   ```
   aws.s3.bucket-user-preferences=<your_s3_bucket_name_for_user_preferences>
   aws.s3.bucket-user-storage=<your_s3_bucket_name_for_user_storage>
   aws.accessKey=<your_aws_access_key>
   aws.secretKey=<your_aws_secret_key>
   aws.region=<your_aws_region>
   aws.cognito.userPoolId=<your_cognito_user_pool_id>
   aws.cognito.clientId=<your_cognito_client_id>
   aws.cognito.clientSecret=<your_cognito_client_secret>
   spring.main.banner-mode=console
   logging.level.root=ERROR
   ```

   Replace the placeholder values with your actual AWS credentials and resource identifiers.

## Dependencies

The project uses the following main dependencies:

- Spring Boot Starter Web
- Spring Boot Starter Security
- AWS Java SDK for Amazon Cognito Identity Provider
- AWS Java SDK for Amazon S3

For a full list of dependencies, please refer to the `pom.xml` file.

## Building

The project uses the Spring Boot Maven plugin for building. The main class is set to `io.github.lwcarani.DropboxCloneApplication`.

To build a JAR file in Eclipse IDE using Maven, follow these steps:

1. Ensure your project is set up as a Maven project in Eclipse.
2. Open the project in Eclipse.
3. Right-click on the project in the Package Explorer.
4. Navigate to "Run As" > "Maven build..."
5. In the "Goals" field, enter "clean package"
6. Click "Run"

Maven will then compile your code, run any tests, and package your application into a JAR file. The JAR file will typically be created in the `target` directory of your project.

## Running the Application

Once you've built a JAR file, you can run the application from the command line, like so:

```
java -jar target\dropbox-clone-0.0.2.jar
```

## Usage

Once the application is running, you can use the following commands. I'll walk through them all later on in this post:

- `signup`: Create a new user account
- `login`: Log in to your account
- `logout`: Log out of your account
- `delete_account`: Permanently delete your account
- `push`: Upload all local files and folders to cloud storage
- `pull`: Download all files and folders from cloud to local machine
- `mkdir <folder_name>`: Create a new directory
- `cd <path>`: Change current directory
- `ls <path>`: List contents of current or specified directory
- `rm <path>`: Delete a directory and its contents (both locally and in cloud)
- `change_root <path>`: Set a new root directory for your Dropbox Clone files
- `help`: Display available commands
- `exit`: Exit the application

## Instructions
To use this as a command-line tool, I recommend adding the JAR file to PATH / system variables. For Windows, create a folder named `Aliases` in your C drive: `C:/Aliases`, and then add this folder to PATH. Next, create a batch file that will execute when you call the specified alias. For example, on my machine, I have a batch file named `dbox.bat` located at `C:/Aliases`, that contains the following script:

```bat
@echo off
echo.
java -jar C:\...\GitHub\dropbox-clone\target\dropbox-clone-0.0.2.jar %*
```

> If you change the version number in the `pom.xml` file, don't forget to also update the version number in this batch file. 
{: .prompt-info }

So now, when I type `dbox` in the command prompt, this batch file will execute, which in turn, runs the `dropbox-clone` application.

## Examples

`dropbox-clone` allows you to execute many commands you would need in a real dropbox application. 

To begin, we make an account for a new user:

```cmd
C:\>dbox

____                   _                  ____ _
|  _ \ _ __ ___  _ __ | |__   _____  __  / ___| | ___  _ __   ___
| | | | '__/ _ \| '_ \| '_ \ / _ \ \/ / | |   | |/ _ \| '_ \ / _ \
| |_| | | | (_) | |_) | |_) | (_) >  <  | |___| | (_) | | | |  __/
|____/|_|  \___/| .__/|_.__/ \___/_/\_\  \____|_|\___/|_| |_|\___|
                |_|
dropbox-clone 0.0.2
Powered by Spring Boot 3.1.0
Welcome to Dropbox Clone CLI!
Type 'help' for a list of commands.
> signup
Enter email address: luke@test.com
Enter username: lwcarani
Enter password:
Confirm password:
Your account has successfully been created!
Please log in with your new credentials.
> 
```

> Note that the application uses the `java.io.Console` package to obfuscate passwords when the user enters them via the command line.
{: .prompt-info }

And now that our account has been created, we can login:

```cmd
C:\>dbox

____                   _                  ____ _
|  _ \ _ __ ___  _ __ | |__   _____  __  / ___| | ___  _ __   ___
| | | | '__/ _ \| '_ \| '_ \ / _ \ \/ / | |   | |/ _ \| '_ \ / _ \
| |_| | | | (_) | |_) | |_) | (_) >  <  | |___| | (_) | | | |  __/
|____/|_|  \___/| .__/|_.__/ \___/_/\_\  \____|_|\___/|_| |_|\___|
                |_|
dropbox-clone 0.0.2
Powered by Spring Boot 3.1.0
Welcome to Dropbox Clone CLI!
Type 'help' for a list of commands.
> signup
Enter email address: luke@test.com
Enter username: lwcarani
Enter password:
Confirm password:
Your account has successfully been created!
Please log in with your new credentials.
> login
Enter username: lwcarani
Enter password:
Welcome back, user!
Login successful. Welcome, lwcarani!
```

Since this is our first time logging in, the application will ask us to specify a root directory. This is the directory where the `dropbox-clone` application will store and sync our folders and files. We'll just specify the `C:\` drive for now. The user can always change this with the `change_root <path>` command. The application also stores this information for the user, so the next time you login, it'll automatically load the last root directory location the user specified:

```cmd
Welcome back, user!
Login successful. Welcome, lwcarani!
Running in headless mode. Please enter the full path for the root directory:
C:/
Root directory set to: C:/
C:\dropbox-clone\lwcarani>
```

Once the user specified `C:\` as the root directory, the application automatically creates the `dropbox-clone\lwcarani` path. This is where any files and folders need to be saved to sync with our cloud storage.

Now we can perform a few different commands. Let's start by making a new directory. You could just go into File Explorer on your machine and start dropping in files and folders, but the application also allows you to make new folders directly in the command line, like so: 

```cmd
C:\dropbox-clone\lwcarani> mkdir foo
Folder created successfully in S3: foo
C:\dropbox-clone\lwcarani> ls
Contents of C:\dropbox-clone\lwcarani:
  foo
C:\dropbox-clone\lwcarani>
```

We see that the folder `foo` was correctly created. 

As an aside, the application will prevent you from taking any action outside of this `C:\dropbox-clone\lwcarani` root path:

```cmd
C:\dropbox-clone\lwcarani> cd C:/
Cannot navigate outside of root directory.
C:\dropbox-clone\lwcarani> rm C:/
Cannot navigate outside of root directory.
C:\dropbox-clone\lwcarani> ls C:/
Cannot navigate outside of root directory.
C:\dropbox-clone\lwcarani>
```

Now, let's try making a few more directories:

```cmd
C:\dropbox-clone\lwcarani> mkdir foo/bar
Folder created successfully in S3: foo/bar
C:\dropbox-clone\lwcarani> mkdir foo/goo
Folder created successfully in S3: foo/goo
C:\dropbox-clone\lwcarani> mkdir foo/zoo
Folder created successfully in S3: foo/zoo
C:\dropbox-clone\lwcarani> mkdir foo/baz
Folder created successfully in S3: foo/baz
C:\dropbox-clone\lwcarani> ls foo
Contents of C:\dropbox-clone\lwcarani/foo:
  bar
  baz
  goo
  zoo
C:\dropbox-clone\lwcarani>
```

Great! Things are looking a little cluttered, let's delete a folder:

```cmd
C:\dropbox-clone\lwcarani> rm foo/goo
Warning: This will delete the directory and all its contents both locally and in S3. Continue? (y/n)
y
Cloud directory deleted successfully: goo
Local directory deleted successfully: goo
C:\dropbox-clone\lwcarani> ls foo
Contents of C:\dropbox-clone\lwcarani/foo:
  bar
  baz
  zoo
C:\dropbox-clone\lwcarani>
```

Now let's push all local folders to the cloud to make sure everything has synced correctly. This is especially useful if you've added folders and files outside of the command line:

```cmd
C:\dropbox-clone\lwcarani> push
Warning: This will overwrite any existing files in the cloud with local files. Continue? (y/n)
y
Push operation started.
Push completed successfully.
C:\dropbox-clone\lwcarani>
```

You can also pull all cloud folders down to your local machine. This is useful if you've uploaded files from one device, and now need to sync all the stored files to another local device:

```cmd
C:\dropbox-clone\lwcarani> pull
Warning: This will overwrite any existing local files with files currently stored in the cloud. Continue? (y/n)
y
Pull operation started.
Pulling from S3 to local root: C:\dropbox-clone\lwcarani
Directory recreated successfully: C:\dropbox-clone\lwcarani\foo
Directory recreated successfully: C:\dropbox-clone\lwcarani\foo\bar
Directory recreated successfully: C:\dropbox-clone\lwcarani\foo\baz
Directory recreated successfully: C:\dropbox-clone\lwcarani\foo\zoo
Pull completed successfully.
C:\dropbox-clone\lwcarani>
```

At any point, you can just logout from the command line when you're finished with your session:

```cmd
C:\dropbox-clone\lwcarani> logout
Logout successful. Goodbye, lwcarani!
>
```

This will also prevent you from conducting any actions that you need to be logged on to perform:

```cmd
> rm C:/
Please login first. Type 'login' to begin.
> mkdir test
Please login first. Type 'login' to begin.
> push
Please login first. Type 'login' to begin.
> pull
Please login first. Type 'login' to begin.
> cd test
Please login first. Type 'login' to begin.
>
```

If you forget what your options are, you can always type `help` to get a list of the available commands:

```cmd
C:\dropbox-clone\lwcarani> help
Available commands:
  signup - Sign up for a new account
  login - Log in to your account
  logout - Log out of your account
  delete_account - Permanently delete your account
  push - Upload all local files and folders to cloud storage
  pull - Download all file files and folders from cloud to local machine
  mkdir <folder_name> - Make a new directory at the specified location
  cd <path> - Change current directory to the specified path
  ls <path> - Display contents of current folder or specified path
  rm <path> - Delete a directory and its contents both locally and from cloud
  chang_root <path> - Set a new root directory for your dropbox-clone files
  help - Show this help message
  exit - Exit the application
C:\dropbox-clone\lwcarani>
```

If you ever want to delete your account, simply login, then use the `delete_account` command:

```cmd
C:\dropbox-clone\lwcarani> delete_account
Warning: proceeding will permanently delete your account and you will lose all files backed up in the cloud. Continue? (y/n)
y
Your account has successfully been deleted!
> login
Enter username: lwcarani
Enter password:
Error during authentication: Incorrect username or password.
Login failed. Please check your credentials. To make a new account, type 'signup' to begin.
>
```

And finally, when you're done with the application, simply enter `exit`, and you will return back to the standard command line prompt:

```cmd
> exit
Goodbye!

C:\>
```

The full code for this project can be found at my Github [here](https://github.com/lwcarani/dropbox-clone).

## Acknowledgements
Thanks to [John Crickett](https://github.com/JohnCrickett) for the idea from his site, [Coding Challenges](https://codingchallenges.fyi/challenges/challenge-dropbox/)!

If you happen to peruse my code and notice any bugs or opportunities for optimizations, please let me know!
