# Attention: This is work in progress and not updated, yet!

# Singularity container for Abaqus 2019

Abaqus 2019, and reportedly 2018, do not run properly under Ubuntu 18.04 LTS. You can start CAE, set up and run a job, but there is a problem with the flexnet license manager, as the license is not returned and thus I had to restart my machine before i could run a second job. Further reads on the issue can be found at:
https://askubuntu.com/questions/1062058/process-hangs-before-termination-with-ubuntu-18-04
https://github.com/imirzov/Install-Abaqus-2019-on-Ubuntu-18.04-LTS

Following a tutorial by Will Furnass https://learningpatterns.me/posts-output/2018-01-30-abaqus-singularity/, I managed to update his workflow to work for the combination of ABQ2019, singularity 3.5, and Ubuntu 18.04. LTS using a centos 7 container, as he did.

## Preparations

1. Make sure you have Singularity 3.5 installed.
2. Symlink the `.iso` installer for Abaqus 2017 into a clone of this repository.
3. Ensure that the license server(s) listed in `assets/cae-UserIntentions_CODE.xml` (see `licenseServer1`/`licenseServer2`/`licenseServer3` in that file) are accessible 
   i.e. if they are only reachable after bringing up a VPN connection, then bring that up now.
4. If `/tmp` is 'small' (e.g. 'only' 16GB) then Singularity will run out of space when trying to build your Abaqus image, 
   so you may want to create a directory somewhere else for Singularity to write temporary files to:

    ```bash
    mkdir -p -m 0700 $HOME/.cache/singularity 
    ```
## Building the image

Right, let's start building our Singularity image.  The build process as defined in our `.def` file:

1. Unpacks the installer ISO;
2. Installs install-time and run-time OS dependencies.  The run-time dependencies include VirtualGL and libjpeg-turbo.
3. Kicks off the Abaqus installer, using 'UserIntentions' files to automate the install.

```bash
sudo SINGULARITY_TMPDIR=$HOME/.cache/singularity singularity build ~/imgs/sing/abaqus-2017-centos-7.img abaqus-2017-centos-7.def
```

## Running the container

To run Abaqus CAE with (NVIDIA) hardware-accelerated graphics:

```
singularity run --nv ~/imgs/sing/abaqus-2017-centos-7.img 
```

which is equivalent to:

```
singularity exec --nv ~/imgs/sing/abaqus-2017-centos-7.img vglrun abaqus cae
```

I need to prefix the above command(s) with `optirun` to ensure I use my NVIDIA GPU and not my on-board graphics chip.

Alternatively, to run Abaqus without hardware-accelerated graphics:

```bash
singularity exec --nv ~/imgs/sing/abaqus-2017-centos-7.img abaqus cae -mesa
```

## How the 'UserIntentions' files were generated

These were produced by going through the Abaqus install process interactively within a Singularity 'sandbox' (writable directory-based container):

```bash
pushd some/dir
tar xfp Abaqus2017.iso
popd

# Build a writable container in a directory called 'centos-7-abaqus-2017-singuarity-sandbox'
sudo singularity build --sandbox centos-7-abaqus-2017-singuarity-sandbox  docker://centos:7

# Get a shell in that container
sudo singularity shell --bind some/dir:/iso-unpacked another/dir:/cfgs centos-7-abaqus-2017-singuarity-sandbox

# Within the container...
rpm --import /etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7
yum install -y ksh redhat-lsb-core perl && yum clean all
cd /iso-unpacked
# Run the Abaqus installer
ksh 1/StartTUI.sh
# Find the '*UserIntentions* files generated during the install process and save them outside the container
```
