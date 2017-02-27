---
date: 2017-02-26T18:41:27-05:00
slug: tensorflow-on-aws
title: Running Tensorflow on AWS GPUs
---


I've been spending some time learning deep learning and tensorflow
recently, and as part of that project I wanted to be able to train
models using GPUs on EC2. This post contains some notes on what it
took to get that working. As many people have commented, the
environment setup is often the hardest part of getting a deep learning
setup going, so hopefully this will be useful reference to someone.

The end result of my work was a [packer image definition][packer] that
builds an AMI that boots an Ubuntu on an ec2 `g2.2xlarge` with all the
appropriate drivers and CUDA and cuDNN installed, and a GPU-enabled
tensorflow preinstalled into a virtualenv.

I'm not going to make my AMIs themselves public, since this is a side
project and I don't want anyone accidentally depending on my AWS
account, but the packer script should Just Work if you `packer build`
it with appropriate credentials.

I probably could have used someone else's AMI and saved myself a lot
of pain, but I have a near-obsessive need to understand how things
work under the hood and Do It Myself.

The rest of this post is some gotchas I learned, that will hopefully
help someone else.

## Installing the kernel driver

Installing CUDA requires two components; A kernel driver, and the
userspace libraries.

The [CUDA installer][cuda] includes a version of the kernel
driver. However, the GRID card in the `g2.2xlarge` instances isn't
supported by the newest NVIDIA driver, so one has to install the
driver separately. Amazon has
[documentation on finding a driver][aws], but the
[NVIDIA search page][nvidia-search] it linked gave me a version that
told me it was too new when I actually tried to install it. It pointed
me at the `367.xx` series, which ended up working.

NVIDIA's installers all seem to support automated install modes, but
they aren't always easy to discover. `--accept-license --no-questions
--ui=none` was the secret here.

Building any kernel module requires kernel headers, and requires that
you build the module for the kernel you're going to be running. On
Ubuntu you can install `linux-headers-generic` and
`linux-headers-virtual` to ensure you always have the latest
headers. However, if you're not also *running* the latest kernel,
you'll be out of luck.

In general (and this is the technique my packer build uses), I find
the best luck by:

- Installing `linux-headers-virtual`
- Upgrading the kernel (`linux-image-virtual`)
- Rebooting the machine (to run the latest kernel)
- Explicitly installing `linux-headers-$(uname -r)` to be sure you
  have the right headers.
- Installing the kernel module

If you ever upgrade kernels again, you'll likely need to reinstall the
driver. On ec2 I solve this by rebuilding the AMI if I need a new
kernel.

One other gotcha is that, on ec2, you'll also need the
`linux-image-extra-*` package. This contains kernel drivers not
normally needed in a virtualized environment. However, that includes
display drivers and their supporting subsystems, and so your modules
will fail to load without it.

Also, by default, once you've got the `extra-` package, the system
will try to load the upstream `nouveau` driver, which will grab the
GPU before NVIDIA's driver can. To prevent that, we need to
[install a blacklist file](https://github.com/nelhage/tf-experiments/blob/242785f760e6b6e1a9c61a628e41f6002d42631b/cloud/scripts/cuda.sh#L35-L42)

## Installing CUDA

The [CUDA installer][cuda] can be downloaded from NVIDIA's
website. Like the driver, the installer can be used in a headless
mode, but of course it has a different inscrutable set of options. I
found the combination I wanted was the oxymoronic `--toolkit --silent
--verbose`; `--toolkit` tells it to install the libraries and
development toolchain; `--silent` actually means "headless mode", and
`--verbose` makes it direct output to stdout, as well as a log file
(so that it shows up in the packer output).

I also pass an explicit path for the toolkit,
`--toolkitpath=/usr/local/cuda-8.0`. A CUDA upgrade broke me when
NVIDIA changed the default path from `/usr/local/cuda` to
`/usr/local/cuda-8.0`. I tried `--toolkitpath=/usr/local/cuda`, but
the installer appears to blacklist that. So now I use `cuda-8.0`,
explicitly, for future-proofing, and create a symlink.

## Installing cuDNN

[cuDNN][cudnn] requires an NVIDIA developer login, which is free but
requires you answer a lot of silly questions about what you're going
to use CUDA and cuDNN for.

Unfortunately, the download links don't work unless you're logged in,
which is a problem for automated installation. I mirrored the cuDNN
installer into an S3 bucket for automated fetching.

Once you've got it, the `cuDNN` installer works yet again differently
from the previous installers; It's a tarball that can be unpacked
directly into `/usr/local` on top of your cuda installation.

## Installing tensorflow

Thanks to recent improvements in Python packaging, and some careful
work on Google's part, installing tensorflow itself is the easy
part. If you've got a `virtualenv` with a recent `pip`, `pip install
tensorflow-gpu` is good enough!

Per the
[tensorflow docs](https://www.tensorflow.org/install/install_linux#the_url_of_the_tensorflow_python_package)
you can also provide a direct link to a `whl` file hosted by Google,
so that you don't depend on PyPI to find the package. I found this
necessary for my install to work in TravisCI, for whatever reason.

# Conclusion

NVIDIA makes available pre-built AMIs that probably would work, and
there are [various services](https://www.floydhub.com/) working on
providing easier hosted tensorflow environments. So, if you're lucky,
none of this will ever be relevant to you.

But various people still have need to install these tools themselves,
or even onto physical hardware. So hopefully this experience report
will help someone. Let me know if any of this advice is useful!

[packer]: https://github.com/nelhage/tf-experiments/blob/master/cloud/ec2-gpu.json
[cuda]: https://developer.nvidia.com/cuda-downloads
[cudnn]: https://developer.nvidia.com/rdp/cudnn-download
[aws]: http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/accelerated-computing-instances.html#gpu-operating-systems
[nvidia-search]: http://www.nvidia.com/Download/Find.aspx
