---
title: "Local Oscap Testing"
date: 2022-08-17T12:49:41-05:00
draft: false
type: post
image: "/images/the-stig.jpeg"
---

# Overview
In the course of our duties at Platform 1 when hardening images, the scanners will output CCEs indicating a STIG failure. When attempting to remediate OpenSCAP findings it is less than optimal to have to commit code and wait for all of the CI pipeline steps before scanning and subsequent failure.
What if you don't have an RHEL license or don't want to bother standing up a RHEL/Oracle Linux/Centos host so you can use `oscap-podman`. Instead, wouldn't it be great if you could perform OpenSCAP scans in a local container and quickly iterate before pushing your changes?

For the purpose of this article I'll use UBI8.

## What You'll Need
- An intense desire for faster feedback loops
- Your application image based on Redhat's Universal Base Image

## How To
Attach to your UBI8 image

`docker run -ti --rm --entrypoint=bash --user 0 <my-ubi8-based-image>`

Install the necessary tools

```
dnf install -y openscap-scanner bzip2 wget unzip
-or-
microdnf install -y openscap-scanner bzip2 wget unzip
```

Download and unzip the same STIG version in use by Iron Bank pipelines

```
wget https://github.com/ComplianceAsCode/content/releases/download/v0.1.62/scap-security-guide-0.1.62-oval-5.10.zip && \
unzip scap-security-guide-0.1.62-oval-5.10.zip
```

### Execute oscap

#### All rules
```
oscap xccdf eval --verbose ERROR --fetch-remote-resources --profile "xccdf_org.ssgproject.content_profile_stig" --results compliance_output_report.xml --report report.html "scap-security-guide-0.1.62-oval-5.10/ssg-rhel8-ds.xml"
```
 
#### A single rule
```
oscap xccdf eval --verbose ERROR --fetch-remote-resources --profile "xccdf_org.ssgproject.content_profile_stig" --rule xccdf_org.ssgproject.content_rule_file_ownership_binary_dirs "scap-security-guide-0.1.62-oval-5.10/ssg-rhel8-ds.xml"
```

Scan results will show Pass or Fail.

Note: If you encounter certificate errors running either of the above commands, you may need to remove any port forwards from your docker run command.

## Remediation Scripts
Most oscap findings will have remediation scripts in the XML schema (e.g. ssg-rhel8-ds.xml) with a system key value of `urn:xccdf:fix:script:sh` but I often find it easier to search within the HTML (for example: https://static.open-scap.org/ssg-guides/ssg-rhel8-guide-rht-ccp.html )

Normally I will use a slightly altered version of the remediation script to find the files triggering the finding, and then examine those file for ownership and/or permission issues so I can better understand how my Dockerfile instructions resulted in oscap findings. Did I chmod 777 when I shouldn't have? Did I chown system libs for a user other than root? Did I lay down files from a tarball without examining what ownership and permission settings I was inheriting?

## Permissions

### RPMS
If you want to see what permissions an rpm is going to bring you can do this either from macOS (brew install rpm) or a UBI container where we see (for example) the auth_pam_tool will have the SUID set so other users can run that file as root:

```
❯ rpm -qpivl mariadb-server.rpm | grep -i auth_pam_tool
warning: mariadb-server.rpm: Header V4 DSA/SHA1 Signature, key ID 1bb943db: NOKEY
lrwxrwxrwx    1 root     root                       66 Feb 10 17:53 /usr/lib/.build-id/16/8ed39e2ad2311e19a9b830624517cf680d9ab5 -> ../../../../usr/lib64/mysql/plugin/auth_pam_tool_dir/auth_pam_tool
drwx------    2 root     root                        0 Feb 10 17:50 /usr/lib64/mysql/plugin/auth_pam_tool_dir
-rwsr-xr-x    1 root     root                    12472 Feb 10 17:52 /usr/lib64/mysql/plugin/auth_pam_tool_dir/auth_pam_tool
```

### Tarballs
Often my team will pull in external source code compressed as `.tgz` or `tar.gz`. When those archives are extracted they bring with them (unless explicitly overriden at extract time) all of the permissons from the source filesystem (eg. owner, group and mode). So don't blindly extract archives onto your filesystem, but instead `chown` and `chmod` as necessary.

As a reminder:

SUID is a special file permission for executable files which enables other users to run the file with effective permissions of the file owner. Instead of the normal x which represents execute permissions, you will see an s (to indicate SUID) special permission for the user.

SGID is a special file permission that also applies to executable files and enables other users to inherit the effective GID of file group owner. Likewise, rather than the usual x which represents execute permissions, you will see an s (to indicate SGID) special permission for group user.