# CentOS 7 STIG Packer Build
Packer-based build files for creating a STIGed CentOS 7 image.

# CI Pipeline
A docker image is built used contains the neccessary libraries such as Packer, Qemu and OpenStack-cli. The build executes inside of a Docker container that will be spun up on a GitLab CI runner (also running in a Docker container).

# Packer template logic
The `centos7.json` Packer template is configured to create a CentOS 7 image with STIGs applied from the ground up. It does the following:
* Instantiate a Qemu guest VM from a CentOS 7 iso with the settings from a kickstart file.
  * This is necessary in order to setup the various partitions required by the STIG guidelines.
* Runs an Ansible playbook that references the following roles:
  * STIG role to harden the image based on DISA standards.
  * cloud-enable role to get the image ready to be used in a cloud environment.
* Uploads the generated image into OpenStack. Application credentials are used for this stage when the build runs on Gitlab. These values can be found in repository -> settings -> CI/CD -> variables.

---

# Prerequisites
* Python
  * Refer to `requirements.txt` for python required modules
* Ansible
* Packer
* qemu system packages (install thru yum or apt etc.)
* Your username in the `kvm` group. If not you'll need to run Packer with root.
  * `sudo usermod -a -G kvm ${USERNAME}`
* An environment variable called `PACKER_SSH_PASSWORD` with the password that Packer should use to log into the guest.  The centos7.json template is set to use the `root` user's password which is set in the kickstart file. To get this you'll need to decrypt the `ansible/vars/all/secrets.yml` file with `ansible-vault`. To get the key ask one of the CPAAC security members.
  * You can also define your own user and password to use by setting the `ssh_username` and `ssh_password` variables in the `centos7.json` Packer template to match the root or whatever user you define in the `ks.cfg` kickstart file.
* OpenStack authentication environment variables. For local packer runs, source these from openrc.sh. For CI/CD runs, you can add them as CI/CD variables.
* 25GB of local storage space (CentOS7 ISO is 10.2GB in size)

# Running Packer
`packer build centos7.json`

# Console access to VM spun up by Packer
When Packer spins up the VM to install the operating system on, by default there won't be a console for you to see what's going on inside of the VM.  To get a VNC console connected to the VM follow these steps:
* On remote host where Packer will run (the bare metal runner):
  * Find the Docker network IP for the container that is running the Packer build process with `docker inspect ${CONTAINER_NAME}`.  Reference the "NetworkSettings" -> "IPAddress" value from the output.  NOTE: the container will only be alive while a build is running.
* On local host
  * Setup a tunnel to the remote host where Packer is running with the IP found from the previous section. NOTE: the port 5957 is setup in the `centos7_ci.json` file to be consistent.
    * `ssh -L :5957:${CONTAINER_IP}:5957 ${HOST}`
  * Use your CentOS dev instnaces' Remote desktop Viewer (Utilities -> Remote Desktop Viewer). If you're not on a CentOS instance find the equivalent on your host or a VNC client.
    * Setup a connection to your local port that is being forwarded to the remote host and use `vnc://localhost:5957` for the Host value.
  
When Packer spins up the VM, it sets up a VNC console locally to wherever its running.  Use this as a window into what's happening inside of the VM, but try not to interfere with the build.  The Packer run output will print out the port it used, this will change every run but you can set it to a static value in the Packer template (`centos7_[ci|local].json`) if you'd like with the following [options](https://www.packer.io/docs/builders/qemu#vnc_bind_address):
```
vnc_bind_address
vnc_port_min
vnc_port_max
```

# Uploading to OpenStack
The Packer template in this repository automatically uploads the produced image into OpenStack in the `post-processors` stage. Make sure to setup the proper OpenStack auth environment variables described above prior to running Packer.

# Ansible STIG Role
The role used to STIG is MindPointGroup's [RHEL7-STIG](https://github.com/MindPointGroup/RHEL7-STIG/tree/master)

# Debug Logging
This will create a `packer.log` that will contain packer messages

`PACKER_LOG=1 PACKER_LOG_PATH=packer.log packer build centos7.json`

# Making Hashed Passwords
This is a handy python3 one-liner that will generate a hashed password. The hash method has to be one of:
* 1 - MD5
* 2a - Blowfish
* 5 - SHA-256
* 6 - SHA-512

`python3 -c "import crypt; print(crypt.crypt(input('clear text password: '), '$' + input('hash method: ') + '$' + input('salt: ')))"`

Alternatively you can ask for a randon generated salt and specifying the hash method from the `crypt` library's list:

`# > python3 -c "import crypt; print(crypt.methods)"
[<crypt.METHOD_SHA512>, <crypt.METHOD_SHA256>, <crypt.METHOD_MD5>, <crypt.METHOD_CRYPT>]`

As an example, this will generate a SHA-512 hashed password:

`python3 -c "import crypt; print(crypt.crypt(input('clear text password: '), crypt.mksalt(crypt.METHOD_SHA512)))"`
