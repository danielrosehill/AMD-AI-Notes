# The PyTorch + ROCM Stack

Working with AI invariably involves working with Python. 

And working with Python invariably involves dealing with virtual environments for standardizing stacks and package consistency. 

Anyone exploring this area for the first time will quickly discover that AI Python libraries can be broadly divided into the following two categories:

- Massive foundational packages like PyTorch 
- Much smaller pip packages like openai

The difference in file size between these two is quite ginormous. 

You can find some PyTorch stacks that are 20 GB plus and pip packages that are just a few MBs.

If you're an old schooler like me, you may recoil in horror at the thought of having to repetitively download these huge packages in different environments. I have an ongoing and totally unreasonable fear of running out of disk space due to creating these environments. 

But neurotic fears aside, pulling down the same images over and over again is also just kind of inefficient and wasteful. 

In `Conda or Docker?` I'll share some notes and observations assisted by Claude on the pros and cons of both.

However, if you're doing local AI on AMD, then you're going to almost certainly want to have PyTorch installed. 

## AMD Docs

AMD maintains some really useful documentation for installing various libraries. Their notes on installing PyTorch for ROCm are here: 

https://rocm.docs.amd.com/projects/install-on-linux/en/develop/install/3rd-party/pytorch-install.html

## Docker Layering 

Before reluctantly accepting that Docker was a good way of running ROCM (nothing against Docker, just seemed like the wrong solution for desktop stuff), I spent a lot of time trying to create and recreate Conda environments. 

Conda offers an elegant solution for folks like me who want to avoid pulling and repulling foundational AI images. However, I also formed the impression over time that Conda is best left to those who need extremely rigidly defined environments, such as those involved in scientific computing. 

If you're just someone like me who simply wants to get PyTorch working on ROCm/AMD, I would advocate going with AMD's recommended method of Docker. It supports robust abstraction and—what took me way too long to figure out—is every bit as useful as Conda for building stacks based on a common foundation.

# Host ROCM And Docker

You can and should have ROCM Installed both on your host and in Docker. 

My recommended day one activity would be pulling a Pytorch + ROCM image and then validating that it "works". 

I'd also recommend version controlling the Docker compose that you use to pull this and bring it up. It's also worth considering adding a watchtower hook so that it automatically updates. 

However, this should be done with consideration. Blindly auto-updating PyTorch ROCm on AMD is kind of a bold move. If you're struggling to get stacks to work and be compatible, the better advice is probably to handle the updates manually, at least initially. 
