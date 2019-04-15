AXE deterministic builds
==========================

<img src="https://raw.githubusercontent.com/AXErunners/media/master/etc/axe-gitian-mojave.png" width="425">

This is a deterministic build environment for [AXE](https://github.com/AXErunners/axe) that uses [Gitian](https://gitian.org/).

Gitian provides a way to be reasonably certain that the AXE executables are really built from the exact source on GitHub and have not been tampered with. It also makes sure that the same, tested dependencies are used and statically built into the executable.

Multiple developers build from source code by following a specific descriptor ("recipe"), cryptographically sign the result, and upload the resulting signature. These results are compared and only if they match is the build is accepted.

More independent Gitian builders are needed, which is why this guide exists.

Requirements
------------

6GB of RAM, at least 2 cores, [Git](https://git-scm.com/), [GnuPG](https://www.gnupg.org/).

It relies upon [Vagrant](https://www.vagrantup.com/) and [VirtualBox](https://www.virtualbox.org/) plus [Ansible](https://www.ansible.com/).

#### VirtualBox

If you use Linux, we recommend obtaining _VirtualBox_ through your package manager instead of the Oracle website.

    sudo apt-get install virtualbox

Linux kernel headers are required to setup the `/dev/vboxdrv` device and VirtualBox kernel module via `virtualbox-dkms`.

#### Vagrant

Download the latest version of _Vagrant_ from [their website](https://www.vagrantup.com/downloads.html) or

    sudo apt-get install vagrant

#### Ansible

Install prerequisites first: `sudo apt-get install build-essential libssl-dev libffi-dev python python-dev python-pip`. Then run:

    sudo pip install -U ansible

#### GnuPG

    sudo apt install gnupg2

##### Apple SDK

[Apple SDK](https://github.com/AXErunners/axe/blob/master/doc/README_osx.md) required for macOS builds. Place this tarball (`MacOSX10.11.sdk.tar.gz`) into _axe-gitian_ folder and _Ansible_ will copy it during the run.

How to get started
------------------

### Decide on a gpg keypair to use for Gitian

You can generate a keypair specifically for Gitian builds with a command like the one below.

```
$ gpg2 --quick-gen-key --batch --passphrase '' "Korben Dallas (axe-gitian) <kdallas@multipass.gov>"
gpg: key 3F0C2117D53A4A49 marked as ultimately trusted
gpg: directory '/home/kdallas/.gnupg/openpgp-revocs.d' created
gpg: revocation certificate stored as '/home/kdallas/.gnupg/openpgp-revocs.d/3F14A629C06FA31D59C64FE93F0C2117D53A4A49.rev'
```

Some explanation of the arguments used in the above example:

    --quick-generate-key --batch   This combination of options allows options to be given on the
                                   command line. Other key generation options use interative
                                   prompts.

    --passphrase ''                Passphrase for the generated key. An empty string as shown here
                                   means save the private key unencrypted.

    "Name (Comment) <Email>"       The user id (also called uid) to associate with the generated
                                   keys. Concatenating a name, an optional comment, and an email
                                   address using this format is a gpg convention.


You can check that the key was generated and added to your local gpg key database, and see its
fingerprint value, like this:
```
$ gpg2 --list-keys
/home/kdallas/.gnupg/pubring.kbx
----------------------------------
pub   rsa2048 2018-04-23 [SC] [expires: 2020-04-22]
      3F14A629C06FA31D59C64FE93F0C2117D53A4A49
uid           [ultimate] Korben Dallas (axe-gitian) <kdallas@multipass.gov>
sub   rsa2048 2018-04-23 [E]
```

Update the `gpg_key_id` and `gpg_key_name` entries in `gitian.yml` as follows:

- `gpg_key_id`: In the example output shown here, this is the 40 character string
`3F14A629C06FA31D59C64FE93F0C2117D53A4A49`. Some versions of gpg may truncate this value, e.g. to 8
or 16 characters. You should be able to use the truncated value.

- `gpg_key_name`: the part before the @ symbol of the associated email address. In our example
this is `kdallas`.

### Edit settings in gitian.yml

```yaml
# URL of repository containing AXE source code.
axe_git_repo_url: 'https://github.com/AXErunners/axe'

# Specific tag or branch you want to build.
axe_version: 'master'

# The name@ in the e-mail address of your GPG key, alternatively a key ID.
gpg_key_name: 'F16219F4C23F91112E9C734A8DFCBF8E5A4D8019'

# OPTIONAL set to import your SSH key into the VM. Example: id_rsa, id_ed25519. Assumed to reside in ~/.ssh
ssh_key_name: ''
```

Make sure all dependencies are installed, and then run:

    vagrant up --provision axe-build

This will provision a Gitian host virtual machine that uses a container (Docker) guest to perform the actual builds.

Use `git stash` to save one's local customizations to `gitian.yml`.

Building AXE
--------------

    vagrant ssh axe-build
    #replace $SIGNER and $VERSION to match your gitian.yml
    ./gitian-build.py --setup $SIGNER $VERSION
    ./gitian-build.py -B $SIGNER $VERSION

The output from `gbuild` is informative. There are some common warnings which can be ignored, e.g. if you get an intermittent privileges error related to LXC then just execute the script again. The most important thing is that one reaches the step which says `Running build script (log in var/build.log)`. If not, then something else is wrong and you should let us know.

Take a look at the variables near the middle of `~/gitian-build.py` and get familiar with its functioning, as it can handle most tasks.

It's also a good idea to regularly `git pull` on this repository to obtain updates and re-run the entire VM provisioning for each release, to ensure current and consistent state for your builder.

Generating and uploading signatures
-----------------------------------

After the build successfully completes, `gsign` will be called. Commit and push your signatures (both the .assert and .assert.sig files) to the [AXErunners/gitian.sigs](https://github.com/AXErunners/gitian.sigs) repository, or if that's not possible then create a pull request.

Signatures can be verified by running `gitian-build.py --verify`, but set `build=false` in the script to skip building. Run a `git pull` beforehand on `gitian.sigs` so you have the latest. The provisioning includes a task which imports AXE developer public keys to the Vagrant user's keyring and sets them to ultimately trusted, but they can also be found at `contrib/gitian-keys` within the AXE source repository.

Working with GPG and SSH
--------------------------

We provide two options for automatically importing keys into the VM, or you may choose to copy them manually. Keys are needed A) to sign the manifests which get pushed to [gitian.sigs](https://github.com/AXErunners/gitian.sigs) and B) to interact with GitHub, if you choose to use an SSH instead of HTTPS remote. The latter would entail always providing your GitHub login and [access token](https://github.com/settings/tokens) in order to push from within the VM.

Your local SSH agent is automatically forwarded into the VM via a configuration option. If you run ssh-agent, your keys should already be available.

GPG is trickier, especially if you use a smartcard and can't copy the secret key. We have a script intended to forward the gpg-agent socket into the VM, `forward_gpg_agent.sh`, but it is not currently working. If you want your full keyring to be available, you can use the following workaround involving `sshfs` and synced folders:

    vagrant plugin install vagrant-sshfs

Uncomment the line beginning with `gitian.vm.synced_folder "~/.gnupg"` in `Vagrantfile`. Ensure the destination mount point is empty. Then run:

    vagrant sshfs --mount axe-build

Vagrant synced folders may also work natively with `vboxfs` if you install VirtualBox Guest Additions into the VM from `contrib`, but that's not as easy to setup.


Copying files
-------------

The easiest way to do it is with a plugin.

    vagrant plugin install vagrant-scp

To copy files to the VM: `vagrant scp file_on_host.txt :file_on_vm.txt`

To copy files from the VM: `vagrant scp :file_on_vm.txt file_on_host.txt`

Other notes
-----------

Port 2200 on the host machine should be forwarded to port 22 on the guest virtual machine.

The automation and configuration management assumes that VirtualBox will assign the IP address `10.0.2.15` to the Gitian host Vagrant VM.

Tested with VirtualBox 6.0, Ansible 2.7.10 and Vagrant 2.2.4 on Ubuntu 18, macOS Mojave.
