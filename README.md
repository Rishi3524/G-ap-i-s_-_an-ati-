## Labs 1-

```
Student ID: 24754678
```
Student Name: Lyuchen Dai


# Lab 1

## AWS Account and Log in

### [1] Log into an IAM user account created for you on AWS.

Go to the portal at https://489389878001.signin.aws.amazon.com/console since our IAM user accounts are
under the AWS account with ID 489389878001.
At the first time you login, you will be prompted to change your password. After logging in, you should see
the AWS Management Console.

### [2] Search and open Identity Access Management

- Search for IAM


going to be a control plane, I don't give it too much resources. It is just an unprivileged container with
unlimited cores and 1GB memory.

## Install Linux packages

### [1] Install Python

Ubuntu 22.04 LTS does come with Python 3.10 pre-installed, so I don't need to install it again. Just install the
pip3 package manager.
**Command:**

```
root in awscli in ~ took 1m7s
⬢ [Systemd] ❯ python3 -V
Python 3.10.
```
**Explaination:**

- -V: is used to print the version of Python installed. Same as --version.
**Command:**

```
root in awscli in ~
⬢ [Systemd] ❯ apt install -y python3-pip
```

