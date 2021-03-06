

# DCGAN Tutorial

**Author**: [Nathan Inkawhich](https://github.com/inkawhich)

## Introduction

This tutorial will give an introduction to DCGANs through an example. We will train a generative adversarial network (GAN) to generate new celebrities after showing it pictures of many real celebrities. Most of the code here is from the dcgan implementation in [pytorch/examples](https://github.com/pytorch/examples), and this document will give a thorough explanation of the implementation and shed light on how and why this model works. But don’t worry, no prior knowledge of GANs is required, but it may require a first-timer to spend some time reasoning about what is actually happening under the hood. Also, for the sake of time it will help to have a GPU, or two. Lets start from the beginning.

## Generative Adversarial Networks

### What is a GAN?

GANs are a framework for teaching a DL model to capture the training data’s distribution so we can generate new data from that same distribution. GANs were invented by Ian Goodfellow in 2014 and first described in the paper [Generative Adversarial Nets](https://papers.nips.cc/paper/5423-generative-adversarial-nets.pdf). They are made of two distinct models, a _generator_ and a _discriminator_. The job of the generator is to spawn ‘fake’ images that look like the training images. The job of the discriminator is to look at an image and output whether or not it is a real training image or a fake image from the generator. During training, the generator is constantly trying to outsmart the discriminator by generating better and better fakes, while the discriminator is working to become a better detective and correctly classify the real and fake images. The equilibrium of this game is when the generator is generating perfect fakes that look as if they came directly from the training data, and the discriminator is left to always guess at 50% confidence that the generator output is real or fake.

Now, lets define some notation to be used throughout tutorial starting with the discriminator. Let `\(x\)` be data representing an image. `\(D(x)\)` is the discriminator network which outputs the (scalar) probability that `\(x\)` came from training data rather than the generator. Here, since we are dealing with images the input to `\(D(x)\)` is an image of HWC size 3x64x64\. Intuitively, `\(D(x)\)` should be HIGH when `\(x\)` comes from training data and LOW when `\(x\)` comes from the generator. `\(D(x)\)` can also be thought of as a traditional binary classifier.

For the generator’s notation, let `\(z\)` be a latent space vector sampled from a standard normal distribution. `\(G(z)\)` represents the generator function which maps the latent vector `\(z\)` to data-space. The goal of `\(G\)` is to estimate the distribution that the training data comes from (`\(p_{data}\)`) so it can generate fake samples from that estimated distribution (`\(p_g\)`).

So, `\(D(G(z))\)` is the probability (scalar) that the output of the generator `\(G\)` is a real image. As described in [Goodfellow’s paper](https://papers.nips.cc/paper/5423-generative-adversarial-nets.pdf), `\(D\)` and `\(G\)` play a minimax game in which `\(D\)` tries to maximize the probability it correctly classifies reals and fakes (`\(logD(x)\)`), and `\(G\)` tries to minimize the probability that `\(D\)` will predict its outputs are fake (`\(log(1-D(G(x)))\)`). From the paper, the GAN loss function is

```py
\[\underset{G}{\text{min}} \underset{D}{\text{max}}V(D,G) = \mathbb{E}_{x\sim p_{data}(x)}\big[logD(x)\big] + \mathbb{E}_{z\sim p_{z}(z)}\big[log(1-D(G(x)))\big]\]
```

In theory, the solution to this minimax game is where `\(p_g = p_{data}\)`, and the discriminator guesses randomly if the inputs are real or fake. However, the convergence theory of GANs is still being actively researched and in reality models do not always train to this point.

### What is a DCGAN?

A DCGAN is a direct extension of the GAN described above, except that it explicitly uses convolutional and convolutional-transpose layers in the discriminator and generator, respectively. It was first described by Radford et. al. in the paper [Unsupervised Representation Learning With Deep Convolutional Generative Adversarial Networks](https://arxiv.org/pdf/1511.06434.pdf). The discriminator is made up of strided [convolution](https://pytorch.org/docs/stable/nn.html#torch.nn.Conv2d) layers, [batch norm](https://pytorch.org/docs/stable/nn.html#torch.nn.BatchNorm2d) layers, and [LeakyReLU](https://pytorch.org/docs/stable/nn.html#torch.nn.LeakyReLU) activations. The input is a 3x64x64 input image and the output is a scalar probability that the input is from the real data distribution. The generator is comprised of [convolutional-transpose](https://pytorch.org/docs/stable/nn.html#torch.nn.ConvTranspose2d) layers, batch norm layers, and [ReLU](https://pytorch.org/docs/stable/nn.html#relu) activations. The input is a latent vector, `\(z\)`, that is drawn from a standard normal distribution and the output is a 3x64x64 RGB image. The strided conv-transpose layers allow the latent vector to be transformed into a volume with the same shape as an image. In the paper, the authors also give some tips about how to setup the optimizers, how to calculate the loss functions, and how to initialize the model weights, all of which will be explained in the coming sections.

```py
from __future__ import print_function
#%matplotlib inline
import argparse
import os
import random
import torch
import torch.nn as nn
import torch.nn.parallel
import torch.backends.cudnn as cudnn
import torch.optim as optim
import torch.utils.data
import torchvision.datasets as dset
import torchvision.transforms as transforms
import torchvision.utils as vutils
import numpy as np
import matplotlib.pyplot as plt
import matplotlib.animation as animation
from IPython.display import HTML

# Set random seem for reproducibility
manualSeed = 999
#manualSeed = random.randint(1, 10000) # use if you want new results
print("Random Seed: ", manualSeed)
random.seed(manualSeed)
torch.manual_seed(manualSeed)

```

Out:

```py
Random Seed:  999

```

## Inputs

Let’s define some inputs for the run:

*   **dataroot** - the path to the root of the dataset folder. We will talk more about the dataset in the next section
*   **workers** - the number of worker threads for loading the data with the DataLoader
*   **batch_size** - the batch size used in training. The DCGAN paper uses a batch size of 128
*   **image_size** - the spatial size of the images used for training. This implementation defaults to 64x64\. If another size is desired, the structures of D and G must be changed. See [here](https://github.com/pytorch/examples/issues/70) for more details
*   **nc** - number of color channels in the input images. For color images this is 3
*   **nz** - length of latent vector
*   **ngf** - relates to the depth of feature maps carried through the generator
*   **ndf** - sets the depth of feature maps propagated through the discriminator
*   **num_epochs** - number of training epochs to run. Training for longer will probably lead to better results but will also take much longer
*   **lr** - learning rate for training. As described in the DCGAN paper, this number should be 0.0002
*   **beta1** - beta1 hyperparameter for Adam optimizers. As described in paper, this number should be 0.5
*   **ngpu** - number of GPUs available. If this is 0, code will run in CPU mode. If this number is greater than 0 it will run on that number of GPUs

```py
# Root directory for dataset
dataroot = "data/celeba"

# Number of workers for dataloader
workers = 2

# Batch size during training
batch_size = 128

# Spatial size of training images. All images will be resized to this
#   size using a transformer.
image_size = 64

# Number of channels in the training images. For color images this is 3
nc = 3

# Size of z latent vector (i.e. size of generator input)
nz = 100

# Size of feature maps in generator
ngf = 64

# Size of feature maps in discriminator
ndf = 64

# Number of training epochs
num_epochs = 5

# Learning rate for optimizers
lr = 0.0002

# Beta1 hyperparam for Adam optimizers
beta1 = 0.5

# Number of GPUs available. Use 0 for CPU mode.
ngpu = 1

```

## Data

In this tutorial we will use the [Celeb-A Faces dataset](https://mmlab.ie.cuhk.edu.hk/projects/CelebA.html) which can be downloaded at the linked site, or in [Google Drive](https://drive.google.com/drive/folders/0B7EVK8r0v71pTUZsaXdaSnZBZzg). The dataset will download as a file named _img_align_celeba.zip_. Once downloaded, create a directory named _celeba_ and extract the zip file into that directory. Then, set the _dataroot_ input for this notebook to the _celeba_ directory you just created. The resulting directory structure should be:

```py
/path/to/celeba
    -> img_align_celeba
        -> 188242.jpg
        -> 173822.jpg
        -> 284702.jpg
        -> 537394.jpg
           ...

```

This is an important step because we will be using the ImageFolder dataset class, which requires there to be subdirectories in the dataset’s root folder. Now, we can create the dataset, create the dataloader, set the device to run on, and finally visualize some of the training data.

```py
# We can use an image folder dataset the way we have it setup.
# Create the dataset
dataset = dset.ImageFolder(root=dataroot,
                           transform=transforms.Compose([
                               transforms.Resize(image_size),
                               transforms.CenterCrop(image_size),
                               transforms.ToTensor(),
                               transforms.Normalize((0.5, 0.5, 0.5), (0.5, 0.5, 0.5)),
                           ]))
# Create the dataloader
dataloader = torch.utils.data.DataLoader(dataset, batch_size=batch_size,
                                         shuffle=True, num_workers=workers)

# Decide which device we want to run on
device = torch.device("cuda:0" if (torch.cuda.is_available() and ngpu > 0) else "cpu")

# Plot some training images
real_batch = next(iter(dataloader))
plt.figure(figsize=(8,8))
plt.axis("off")
plt.title("Training Images")
plt.imshow(np.transpose(vutils.make_grid(real_batch[0].to(device)[:64], padding=2, normalize=True).cpu(),(1,2,0)))

```

![https://pytorch.org/tutorials/_images/sphx_glr_dcgan_faces_tutorial_001.png](img/04fb3a8ed8e63cf7cffb5f29224decca.jpg)

## Implementation

With our input parameters set and the dataset prepared, we can now get into the implementation. We will start with the weigth initialization strategy, then talk about the generator, discriminator, loss functions, and training loop in detail.

### Weight Initialization

From the DCGAN paper, the authors specify that all model weights shall be randomly initialized from a Normal distribution with mean=0, stdev=0.2\. The `weights_init` function takes an initialized model as input and reinitializes all convolutional, convolutional-transpose, and batch normalization layers to meet this criteria. This function is applied to the models immediately after initialization.

```py
# custom weights initialization called on netG and netD
def weights_init(m):
    classname = m.__class__.__name__
    if classname.find('Conv') != -1:
        nn.init.normal_(m.weight.data, 0.0, 0.02)
    elif classname.find('BatchNorm') != -1:
        nn.init.normal_(m.weight.data, 1.0, 0.02)
        nn.init.constant_(m.bias.data, 0)

```

### Generator

The generator, `\(G\)`, is designed to map the latent space vector (`\(z\)`) to data-space. Since our data are images, converting `\(z\)` to data-space means ultimately creating a RGB image with the same size as the training images (i.e. 3x64x64). In practice, this is accomplished through a series of strided two dimensional convolutional transpose layers, each paired with a 2d batch norm layer and a relu activation. The output of the generator is fed through a tanh function to return it to the input data range of `\([-1,1]\)`. It is worth noting the existence of the batch norm functions after the conv-transpose layers, as this is a critical contribution of the DCGAN paper. These layers help with the flow of gradients during training. An image of the generator from the DCGAN paper is shown below.

![dcgan_generator](img/85974d98be6202902f21ce274418953f.jpg)

Notice, the how the inputs we set in the input section (_nz_, _ngf_, and _nc_) influence the generator architecture in code. _nz_ is the length of the z input vector, _ngf_ relates to the size of the feature maps that are propagated through the generator, and _nc_ is the number of channels in the output image (set to 3 for RGB images). Below is the code for the generator.

```py
# Generator Code

class Generator(nn.Module):
    def __init__(self, ngpu):
        super(Generator, self).__init__()
        self.ngpu = ngpu
        self.main = nn.Sequential(
            # input is Z, going into a convolution
            nn.ConvTranspose2d( nz, ngf * 8, 4, 1, 0, bias=False),
            nn.BatchNorm2d(ngf * 8),
            nn.ReLU(True),
            # state size. (ngf*8) x 4 x 4
            nn.ConvTranspose2d(ngf * 8, ngf * 4, 4, 2, 1, bias=False),
            nn.BatchNorm2d(ngf * 4),
            nn.ReLU(True),
            # state size. (ngf*4) x 8 x 8
            nn.ConvTranspose2d( ngf * 4, ngf * 2, 4, 2, 1, bias=False),
            nn.BatchNorm2d(ngf * 2),
            nn.ReLU(True),
            # state size. (ngf*2) x 16 x 16
            nn.ConvTranspose2d( ngf * 2, ngf, 4, 2, 1, bias=False),
            nn.BatchNorm2d(ngf),
            nn.ReLU(True),
            # state size. (ngf) x 32 x 32
            nn.ConvTranspose2d( ngf, nc, 4, 2, 1, bias=False),
            nn.Tanh()
            # state size. (nc) x 64 x 64
        )

    def forward(self, input):
        return self.main(input)

```

Now, we can instantiate the generator and apply the `weights_init` function. Check out the printed model to see how the generator object is structured.

```py
# Create the generator
netG = Generator(ngpu).to(device)

# Handle multi-gpu if desired
if (device.type == 'cuda') and (ngpu > 1):
    netG = nn.DataParallel(netG, list(range(ngpu)))

# Apply the weights_init function to randomly initialize all weights
#  to mean=0, stdev=0.2.
netG.apply(weights_init)

# Print the model
print(netG)

```

Out:

```py
Generator(
  (main): Sequential(
    (0): ConvTranspose2d(100, 512, kernel_size=(4, 4), stride=(1, 1), bias=False)
    (1): BatchNorm2d(512, eps=1e-05, momentum=0.1, affine=True, track_running_stats=True)
    (2): ReLU(inplace)
    (3): ConvTranspose2d(512, 256, kernel_size=(4, 4), stride=(2, 2), padding=(1, 1), bias=False)
    (4): BatchNorm2d(256, eps=1e-05, momentum=0.1, affine=True, track_running_stats=True)
    (5): ReLU(inplace)
    (6): ConvTranspose2d(256, 128, kernel_size=(4, 4), stride=(2, 2), padding=(1, 1), bias=False)
    (7): BatchNorm2d(128, eps=1e-05, momentum=0.1, affine=True, track_running_stats=True)
    (8): ReLU(inplace)
    (9): ConvTranspose2d(128, 64, kernel_size=(4, 4), stride=(2, 2), padding=(1, 1), bias=False)
    (10): BatchNorm2d(64, eps=1e-05, momentum=0.1, affine=True, track_running_stats=True)
    (11): ReLU(inplace)
    (12): ConvTranspose2d(64, 3, kernel_size=(4, 4), stride=(2, 2), padding=(1, 1), bias=False)
    (13): Tanh()
  )
)

```

### Discriminator

As mentioned, the discriminator, `\(D\)`, is a binary classification network that takes an image as input and outputs a scalar probability that the input image is real (as opposed to fake). Here, `\(D\)` takes a 3x64x64 input image, processes it through a series of Conv2d, BatchNorm2d, and LeakyReLU layers, and outputs the final probability through a Sigmoid activation function. This architecture can be extended with more layers if necessary for the problem, but there is significance to the use of the strided convolution, BatchNorm, and LeakyReLUs. The DCGAN paper mentions it is a good practice to use strided convolution rather than pooling to downsample because it lets the network learn its own pooling function. Also batch norm and leaky relu functions promote healthy gradient flow which is critical for the learning process of both `\(G\)` and `\(D\)`.

Discriminator Code

```py
class Discriminator(nn.Module):
    def __init__(self, ngpu):
        super(Discriminator, self).__init__()
        self.ngpu = ngpu
        self.main = nn.Sequential(
            # input is (nc) x 64 x 64
            nn.Conv2d(nc, ndf, 4, 2, 1, bias=False),
            nn.LeakyReLU(0.2, inplace=True),
            # state size. (ndf) x 32 x 32
            nn.Conv2d(ndf, ndf * 2, 4, 2, 1, bias=False),
            nn.BatchNorm2d(ndf * 2),
            nn.LeakyReLU(0.2, inplace=True),
            # state size. (ndf*2) x 16 x 16
            nn.Conv2d(ndf * 2, ndf * 4, 4, 2, 1, bias=False),
            nn.BatchNorm2d(ndf * 4),
            nn.LeakyReLU(0.2, inplace=True),
            # state size. (ndf*4) x 8 x 8
            nn.Conv2d(ndf * 4, ndf * 8, 4, 2, 1, bias=False),
            nn.BatchNorm2d(ndf * 8),
            nn.LeakyReLU(0.2, inplace=True),
            # state size. (ndf*8) x 4 x 4
            nn.Conv2d(ndf * 8, 1, 4, 1, 0, bias=False),
            nn.Sigmoid()
        )

    def forward(self, input):
        return self.main(input)

```

Now, as with the generator, we can create the discriminator, apply the `weights_init` function, and print the model’s structure.

```py
# Create the Discriminator
netD = Discriminator(ngpu).to(device)

# Handle multi-gpu if desired
if (device.type == 'cuda') and (ngpu > 1):
    netD = nn.DataParallel(netD, list(range(ngpu)))

# Apply the weights_init function to randomly initialize all weights
#  to mean=0, stdev=0.2.
netD.apply(weights_init)

# Print the model
print(netD)

```

Out:

```py
Discriminator(
  (main): Sequential(
    (0): Conv2d(3, 64, kernel_size=(4, 4), stride=(2, 2), padding=(1, 1), bias=False)
    (1): LeakyReLU(negative_slope=0.2, inplace)
    (2): Conv2d(64, 128, kernel_size=(4, 4), stride=(2, 2), padding=(1, 1), bias=False)
    (3): BatchNorm2d(128, eps=1e-05, momentum=0.1, affine=True, track_running_stats=True)
    (4): LeakyReLU(negative_slope=0.2, inplace)
    (5): Conv2d(128, 256, kernel_size=(4, 4), stride=(2, 2), padding=(1, 1), bias=False)
    (6): BatchNorm2d(256, eps=1e-05, momentum=0.1, affine=True, track_running_stats=True)
    (7): LeakyReLU(negative_slope=0.2, inplace)
    (8): Conv2d(256, 512, kernel_size=(4, 4), stride=(2, 2), padding=(1, 1), bias=False)
    (9): BatchNorm2d(512, eps=1e-05, momentum=0.1, affine=True, track_running_stats=True)
    (10): LeakyReLU(negative_slope=0.2, inplace)
    (11): Conv2d(512, 1, kernel_size=(4, 4), stride=(1, 1), bias=False)
    (12): Sigmoid()
  )
)

```

### Loss Functions and Optimizers

With `\(D\)` and `\(G\)` setup, we can specify how they learn through the loss functions and optimizers. We will use the Binary Cross Entropy loss ([BCELoss](https://pytorch.org/docs/stable/nn.html#torch.nn.BCELoss)) function which is defined in PyTorch as:

```py
\[\ell(x, y) = L = \{l_1,\dots,l_N\}^\top, \quad l_n = - \left[ y_n \cdot \log x_n + (1 - y_n) \cdot \log (1 - x_n) \right]\]
```

Notice how this function provides the calculation of both log components in the objective function (i.e. `\(log(D(x))\)` and `\(log(1-D(G(z)))\)`). We can specify what part of the BCE equation to use with the `\(y\)` input. This is accomplished in the training loop which is coming up soon, but it is important to understand how we can choose which component we wish to calculate just by changing `\(y\)` (i.e. GT labels).

Next, we define our real label as 1 and the fake label as 0\. These labels will be used when calculating the losses of `\(D\)` and `\(G\)`, and this is also the convention used in the original GAN paper. Finally, we set up two separate optimizers, one for `\(D\)` and one for `\(G\)`. As specified in the DCGAN paper, both are Adam optimizers with learning rate 0.0002 and Beta1 = 0.5\. For keeping track of the generator’s learning progression, we will generate a fixed batch of latent vectors that are drawn from a Gaussian distribution (i.e. fixed_noise) . In the training loop, we will periodically input this fixed_noise into `\(G\)`, and over the iterations we will see images form out of the noise.

```py
# Initialize BCELoss function
criterion = nn.BCELoss()

# Create batch of latent vectors that we will use to visualize
#  the progression of the generator
fixed_noise = torch.randn(64, nz, 1, 1, device=device)

# Establish convention for real and fake labels during training
real_label = 1
fake_label = 0

# Setup Adam optimizers for both G and D
optimizerD = optim.Adam(netD.parameters(), lr=lr, betas=(beta1, 0.999))
optimizerG = optim.Adam(netG.parameters(), lr=lr, betas=(beta1, 0.999))

```

### Training

Finally, now that we have all of the parts of the GAN framework defined, we can train it. Be mindful that training GANs is somewhat of an art form, as incorrect hyperparameter settings lead to mode collapse with little explanation of what went wrong. Here, we will closely follow Algorithm 1 from Goodfellow’s paper, while abiding by some of the best practices shown in [ganhacks](https://github.com/soumith/ganhacks). Namely, we will “construct different mini-batches for real and fake” images, and also adjust G’s objective function to maximize `\(logD(G(z))\)`. Training is split up into two main parts. Part 1 updates the Discriminator and Part 2 updates the Generator.

**Part 1 - Train the Discriminator**

Recall, the goal of training the discriminator is to maximize the probability of correctly classifying a given input as real or fake. In terms of Goodfellow, we wish to “update the discriminator by ascending its stochastic gradient”. Practically, we want to maximize `\(log(D(x)) + log(1-D(G(z)))\)`. Due to the separate mini-batch suggestion from ganhacks, we will calculate this in two steps. First, we will construct a batch of real samples from the training set, forward pass through `\(D\)`, calculate the loss (`\(log(D(x))\)`), then calculate the gradients in a backward pass. Secondly, we will construct a batch of fake samples with the current generator, forward pass this batch through `\(D\)`, calculate the loss (`\(log(1-D(G(z)))\)`), and _accumulate_ the gradients with a backward pass. Now, with the gradients accumulated from both the all-real and all-fake batches, we call a step of the Discriminator’s optimizer.

**Part 2 - Train the Generator**

As stated in the original paper, we want to train the Generator by minimizing `\(log(1-D(G(z)))\)` in an effort to generate better fakes. As mentioned, this was shown by Goodfellow to not provide sufficient gradients, especially early in the learning process. As a fix, we instead wish to maximize `\(log(D(G(z)))\)`. In the code we accomplish this by: classifying the Generator output from Part 1 with the Discriminator, computing G’s loss _using real labels as GT_, computing G’s gradients in a backward pass, and finally updating G’s parameters with an optimizer step. It may seem counter-intuitive to use the real labels as GT labels for the loss function, but this allows us to use the `\(log(x)\)` part of the BCELoss (rather than the `\(log(1-x)\)` part) which is exactly what we want.

Finally, we will do some statistic reporting and at the end of each epoch we will push our fixed_noise batch through the generator to visually track the progress of G’s training. The training statistics reported are:

*   **Loss_D** - discriminator loss calculated as the sum of losses for the all real and all fake batches (`\(log(D(x)) + log(D(G(z)))\)`).
*   **Loss_G** - generator loss calculated as `\(log(D(G(z)))\)`
*   **D(x)** - the average output (across the batch) of the discriminator for the all real batch. This should start close to 1 then theoretically converge to 0.5 when G gets better. Think about why this is.
*   **D(G(z))** - average discriminator outputs for the all fake batch. The first number is before D is updated and the second number is after D is updated. These numbers should start near 0 and converge to 0.5 as G gets better. Think about why this is.

**Note:** This step might take a while, depending on how many epochs you run and if you removed some data from the dataset.

```py
# Training Loop

# Lists to keep track of progress
img_list = []
G_losses = []
D_losses = []
iters = 0

print("Starting Training Loop...")
# For each epoch
for epoch in range(num_epochs):
    # For each batch in the dataloader
    for i, data in enumerate(dataloader, 0):

        ############################
        # (1) Update D network: maximize log(D(x)) + log(1 - D(G(z)))
        ###########################
        ## Train with all-real batch
        netD.zero_grad()
        # Format batch
        real_cpu = data[0].to(device)
        b_size = real_cpu.size(0)
        label = torch.full((b_size,), real_label, device=device)
        # Forward pass real batch through D
        output = netD(real_cpu).view(-1)
        # Calculate loss on all-real batch
        errD_real = criterion(output, label)
        # Calculate gradients for D in backward pass
        errD_real.backward()
        D_x = output.mean().item()

        ## Train with all-fake batch
        # Generate batch of latent vectors
        noise = torch.randn(b_size, nz, 1, 1, device=device)
        # Generate fake image batch with G
        fake = netG(noise)
        label.fill_(fake_label)
        # Classify all fake batch with D
        output = netD(fake.detach()).view(-1)
        # Calculate D's loss on the all-fake batch
        errD_fake = criterion(output, label)
        # Calculate the gradients for this batch
        errD_fake.backward()
        D_G_z1 = output.mean().item()
        # Add the gradients from the all-real and all-fake batches
        errD = errD_real + errD_fake
        # Update D
        optimizerD.step()

        ############################
        # (2) Update G network: maximize log(D(G(z)))
        ###########################
        netG.zero_grad()
        label.fill_(real_label)  # fake labels are real for generator cost
        # Since we just updated D, perform another forward pass of all-fake batch through D
        output = netD(fake).view(-1)
        # Calculate G's loss based on this output
        errG = criterion(output, label)
        # Calculate gradients for G
        errG.backward()
        D_G_z2 = output.mean().item()
        # Update G
        optimizerG.step()

        # Output training stats
        if i % 50 == 0:
            print('[%d/%d][%d/%d]\tLoss_D: %.4f\tLoss_G: %.4f\tD(x): %.4f\tD(G(z)): %.4f / %.4f'
                  % (epoch, num_epochs, i, len(dataloader),
                     errD.item(), errG.item(), D_x, D_G_z1, D_G_z2))

        # Save Losses for plotting later
        G_losses.append(errG.item())
        D_losses.append(errD.item())

        # Check how the generator is doing by saving G's output on fixed_noise
        if (iters % 500 == 0) or ((epoch == num_epochs-1) and (i == len(dataloader)-1)):
            with torch.no_grad():
                fake = netG(fixed_noise).detach().cpu()
            img_list.append(vutils.make_grid(fake, padding=2, normalize=True))

        iters += 1

```

Out:

```py
Starting Training Loop...
[0/5][0/1583]   Loss_D: 1.7410  Loss_G: 4.7761  D(x): 0.5343    D(G(z)): 0.5771 / 0.0136
[0/5][50/1583]  Loss_D: 1.7332  Loss_G: 25.4829 D(x): 0.9774    D(G(z)): 0.7441 / 0.0000
[0/5][100/1583] Loss_D: 1.6841  Loss_G: 11.6585 D(x): 0.4728    D(G(z)): 0.0000 / 0.0000
[0/5][150/1583] Loss_D: 1.2547  Loss_G: 8.7245  D(x): 0.9286    D(G(z)): 0.5209 / 0.0044
[0/5][200/1583] Loss_D: 0.7563  Loss_G: 8.9600  D(x): 0.9525    D(G(z)): 0.4514 / 0.0003
[0/5][250/1583] Loss_D: 1.0221  Loss_G: 2.5713  D(x): 0.5274    D(G(z)): 0.0474 / 0.1177
[0/5][300/1583] Loss_D: 0.3387  Loss_G: 3.8185  D(x): 0.8431    D(G(z)): 0.1066 / 0.0461
[0/5][350/1583] Loss_D: 0.5054  Loss_G: 3.6141  D(x): 0.7289    D(G(z)): 0.0758 / 0.0535
[0/5][400/1583] Loss_D: 0.8758  Loss_G: 6.5680  D(x): 0.8097    D(G(z)): 0.4017 / 0.0031
[0/5][450/1583] Loss_D: 0.2486  Loss_G: 3.5121  D(x): 0.9035    D(G(z)): 0.1054 / 0.0717
[0/5][500/1583] Loss_D: 1.5792  Loss_G: 4.3590  D(x): 0.3457    D(G(z)): 0.0053 / 0.0379
[0/5][550/1583] Loss_D: 0.8897  Loss_G: 3.9447  D(x): 0.5350    D(G(z)): 0.0349 / 0.0386
[0/5][600/1583] Loss_D: 0.5292  Loss_G: 4.4346  D(x): 0.8914    D(G(z)): 0.2768 / 0.0233
[0/5][650/1583] Loss_D: 0.3779  Loss_G: 4.7253  D(x): 0.7868    D(G(z)): 0.0627 / 0.0174
[0/5][700/1583] Loss_D: 0.7512  Loss_G: 2.6246  D(x): 0.6112    D(G(z)): 0.0244 / 0.1493
[0/5][750/1583] Loss_D: 0.4378  Loss_G: 5.0045  D(x): 0.8614    D(G(z)): 0.2028 / 0.0108
[0/5][800/1583] Loss_D: 0.5795  Loss_G: 6.0537  D(x): 0.8693    D(G(z)): 0.2732 / 0.0066
[0/5][850/1583] Loss_D: 0.8980  Loss_G: 6.5355  D(x): 0.8465    D(G(z)): 0.4226 / 0.0048
[0/5][900/1583] Loss_D: 0.5776  Loss_G: 7.7162  D(x): 0.9756    D(G(z)): 0.3707 / 0.0009
[0/5][950/1583] Loss_D: 0.5593  Loss_G: 5.6692  D(x): 0.9560    D(G(z)): 0.3494 / 0.0080
[0/5][1000/1583]        Loss_D: 0.5036  Loss_G: 5.1312  D(x): 0.7775    D(G(z)): 0.0959 / 0.0178
[0/5][1050/1583]        Loss_D: 0.5192  Loss_G: 4.5706  D(x): 0.8578    D(G(z)): 0.2605 / 0.0222
[0/5][1100/1583]        Loss_D: 0.5645  Loss_G: 3.1618  D(x): 0.7133    D(G(z)): 0.1138 / 0.0768
[0/5][1150/1583]        Loss_D: 0.2790  Loss_G: 4.5294  D(x): 0.8541    D(G(z)): 0.0909 / 0.0207
[0/5][1200/1583]        Loss_D: 0.5334  Loss_G: 4.3445  D(x): 0.8567    D(G(z)): 0.2457 / 0.0245
[0/5][1250/1583]        Loss_D: 0.7318  Loss_G: 2.2779  D(x): 0.6846    D(G(z)): 0.1485 / 0.1497
[0/5][1300/1583]        Loss_D: 0.6939  Loss_G: 6.1172  D(x): 0.9123    D(G(z)): 0.3853 / 0.0041
[0/5][1350/1583]        Loss_D: 0.4653  Loss_G: 3.7054  D(x): 0.8208    D(G(z)): 0.1774 / 0.0404
[0/5][1400/1583]        Loss_D: 1.9711  Loss_G: 3.1569  D(x): 0.2704    D(G(z)): 0.0108 / 0.1390
[0/5][1450/1583]        Loss_D: 0.4427  Loss_G: 5.8683  D(x): 0.9230    D(G(z)): 0.2600 / 0.0056
[0/5][1500/1583]        Loss_D: 0.4432  Loss_G: 3.3681  D(x): 0.8001    D(G(z)): 0.1510 / 0.0633
[0/5][1550/1583]        Loss_D: 0.4852  Loss_G: 3.2790  D(x): 0.7532    D(G(z)): 0.1100 / 0.0661
[1/5][0/1583]   Loss_D: 0.3536  Loss_G: 4.5358  D(x): 0.8829    D(G(z)): 0.1714 / 0.0173
[1/5][50/1583]  Loss_D: 0.4717  Loss_G: 4.7728  D(x): 0.8973    D(G(z)): 0.2750 / 0.0142
[1/5][100/1583] Loss_D: 0.4702  Loss_G: 2.3528  D(x): 0.7847    D(G(z)): 0.1468 / 0.1385
[1/5][150/1583] Loss_D: 0.4833  Loss_G: 2.9645  D(x): 0.7893    D(G(z)): 0.1607 / 0.0867
[1/5][200/1583] Loss_D: 0.6035  Loss_G: 2.0728  D(x): 0.6646    D(G(z)): 0.0852 / 0.1806
[1/5][250/1583] Loss_D: 0.3822  Loss_G: 3.1946  D(x): 0.7969    D(G(z)): 0.1024 / 0.0656
[1/5][300/1583] Loss_D: 0.3892  Loss_G: 3.3337  D(x): 0.7848    D(G(z)): 0.0969 / 0.0525
[1/5][350/1583] Loss_D: 1.7989  Loss_G: 7.5798  D(x): 0.9449    D(G(z)): 0.7273 / 0.0011
[1/5][400/1583] Loss_D: 0.4765  Loss_G: 3.0655  D(x): 0.7479    D(G(z)): 0.1116 / 0.0687
[1/5][450/1583] Loss_D: 0.3649  Loss_G: 3.1674  D(x): 0.8603    D(G(z)): 0.1619 / 0.0627
[1/5][500/1583] Loss_D: 0.6922  Loss_G: 4.5841  D(x): 0.9235    D(G(z)): 0.4003 / 0.0175
[1/5][550/1583] Loss_D: 0.6126  Loss_G: 4.6642  D(x): 0.8761    D(G(z)): 0.3199 / 0.0180
[1/5][600/1583] Loss_D: 0.7032  Loss_G: 4.6221  D(x): 0.9463    D(G(z)): 0.4365 / 0.0154
[1/5][650/1583] Loss_D: 0.4707  Loss_G: 3.3616  D(x): 0.7664    D(G(z)): 0.1280 / 0.0617
[1/5][700/1583] Loss_D: 0.3393  Loss_G: 2.4236  D(x): 0.9120    D(G(z)): 0.1771 / 0.1280
[1/5][750/1583] Loss_D: 0.6828  Loss_G: 4.4585  D(x): 0.8647    D(G(z)): 0.3546 / 0.0191
[1/5][800/1583] Loss_D: 0.7958  Loss_G: 3.6708  D(x): 0.8386    D(G(z)): 0.3987 / 0.0403
[1/5][850/1583] Loss_D: 0.4651  Loss_G: 2.7477  D(x): 0.7602    D(G(z)): 0.1334 / 0.0900
[1/5][900/1583] Loss_D: 0.8799  Loss_G: 4.7930  D(x): 0.9050    D(G(z)): 0.4710 / 0.0201
[1/5][950/1583] Loss_D: 0.3909  Loss_G: 2.7973  D(x): 0.7730    D(G(z)): 0.0902 / 0.0838
[1/5][1000/1583]        Loss_D: 0.3822  Loss_G: 3.0223  D(x): 0.8699    D(G(z)): 0.1837 / 0.0709
[1/5][1050/1583]        Loss_D: 0.4689  Loss_G: 2.2831  D(x): 0.7096    D(G(z)): 0.0536 / 0.1448
[1/5][1100/1583]        Loss_D: 0.6676  Loss_G: 2.2773  D(x): 0.6669    D(G(z)): 0.1386 / 0.1443
[1/5][1150/1583]        Loss_D: 0.5970  Loss_G: 4.1558  D(x): 0.9166    D(G(z)): 0.3554 / 0.0240
[1/5][1200/1583]        Loss_D: 0.3622  Loss_G: 3.5782  D(x): 0.8590    D(G(z)): 0.1547 / 0.0481
[1/5][1250/1583]        Loss_D: 0.5234  Loss_G: 2.5915  D(x): 0.7811    D(G(z)): 0.1990 / 0.1037
[1/5][1300/1583]        Loss_D: 1.3243  Loss_G: 5.5428  D(x): 0.9882    D(G(z)): 0.6572 / 0.0088
[1/5][1350/1583]        Loss_D: 0.4891  Loss_G: 1.9552  D(x): 0.7686    D(G(z)): 0.1540 / 0.1910
[1/5][1400/1583]        Loss_D: 0.5639  Loss_G: 3.7796  D(x): 0.9137    D(G(z)): 0.3390 / 0.0343
[1/5][1450/1583]        Loss_D: 1.7329  Loss_G: 5.0373  D(x): 0.9760    D(G(z)): 0.7332 / 0.0161
[1/5][1500/1583]        Loss_D: 0.7999  Loss_G: 3.7268  D(x): 0.9029    D(G(z)): 0.4550 / 0.0384
[1/5][1550/1583]        Loss_D: 0.4740  Loss_G: 2.3220  D(x): 0.7824    D(G(z)): 0.1625 / 0.1327
[2/5][0/1583]   Loss_D: 0.8693  Loss_G: 3.8890  D(x): 0.9376    D(G(z)): 0.4822 / 0.0339
[2/5][50/1583]  Loss_D: 0.3742  Loss_G: 2.5041  D(x): 0.8148    D(G(z)): 0.1310 / 0.1151
[2/5][100/1583] Loss_D: 1.1134  Loss_G: 1.5167  D(x): 0.4248    D(G(z)): 0.0335 / 0.3023
[2/5][150/1583] Loss_D: 0.5987  Loss_G: 3.2047  D(x): 0.8536    D(G(z)): 0.3121 / 0.0555
[2/5][200/1583] Loss_D: 2.0846  Loss_G: 1.5473  D(x): 0.1919    D(G(z)): 0.0054 / 0.2899
[2/5][250/1583] Loss_D: 0.5017  Loss_G: 3.0225  D(x): 0.8965    D(G(z)): 0.2986 / 0.0626
[2/5][300/1583] Loss_D: 1.3296  Loss_G: 4.1927  D(x): 0.9444    D(G(z)): 0.6574 / 0.0270
[2/5][350/1583] Loss_D: 0.4905  Loss_G: 2.7693  D(x): 0.8049    D(G(z)): 0.2090 / 0.0863
[2/5][400/1583] Loss_D: 0.4668  Loss_G: 2.1790  D(x): 0.7160    D(G(z)): 0.0815 / 0.1529
[2/5][450/1583] Loss_D: 0.4877  Loss_G: 2.4190  D(x): 0.6943    D(G(z)): 0.0693 / 0.1254
[2/5][500/1583] Loss_D: 0.7856  Loss_G: 2.2362  D(x): 0.6148    D(G(z)): 0.1698 / 0.1489
[2/5][550/1583] Loss_D: 0.6371  Loss_G: 1.3879  D(x): 0.6164    D(G(z)): 0.0852 / 0.3041
[2/5][600/1583] Loss_D: 0.6409  Loss_G: 2.8623  D(x): 0.7658    D(G(z)): 0.2684 / 0.0790
[2/5][650/1583] Loss_D: 0.6454  Loss_G: 1.5708  D(x): 0.6293    D(G(z)): 0.0944 / 0.2706
[2/5][700/1583] Loss_D: 0.8472  Loss_G: 2.0847  D(x): 0.5071    D(G(z)): 0.0181 / 0.1937
[2/5][750/1583] Loss_D: 1.2356  Loss_G: 0.3673  D(x): 0.3606    D(G(z)): 0.0328 / 0.7270
[2/5][800/1583] Loss_D: 0.4852  Loss_G: 2.7325  D(x): 0.8670    D(G(z)): 0.2630 / 0.0877
[2/5][850/1583] Loss_D: 0.6494  Loss_G: 4.5357  D(x): 0.8899    D(G(z)): 0.3756 / 0.0158
[2/5][900/1583] Loss_D: 0.5184  Loss_G: 2.7194  D(x): 0.8377    D(G(z)): 0.2540 / 0.0871
[2/5][950/1583] Loss_D: 0.9771  Loss_G: 4.6200  D(x): 0.9596    D(G(z)): 0.5432 / 0.0176
[2/5][1000/1583]        Loss_D: 0.7509  Loss_G: 2.2864  D(x): 0.5861    D(G(z)): 0.1021 / 0.1539
[2/5][1050/1583]        Loss_D: 0.4512  Loss_G: 3.2484  D(x): 0.8649    D(G(z)): 0.2313 / 0.0542
[2/5][1100/1583]        Loss_D: 0.6856  Loss_G: 2.2425  D(x): 0.6405    D(G(z)): 0.1333 / 0.1508
[2/5][1150/1583]        Loss_D: 0.5271  Loss_G: 3.0327  D(x): 0.8385    D(G(z)): 0.2552 / 0.0639
[2/5][1200/1583]        Loss_D: 0.4058  Loss_G: 2.9557  D(x): 0.8769    D(G(z)): 0.2169 / 0.0694
[2/5][1250/1583]        Loss_D: 0.5564  Loss_G: 2.9065  D(x): 0.8409    D(G(z)): 0.2835 / 0.0695
[2/5][1300/1583]        Loss_D: 0.4703  Loss_G: 2.7865  D(x): 0.7825    D(G(z)): 0.1680 / 0.0850
[2/5][1350/1583]        Loss_D: 0.5352  Loss_G: 3.1362  D(x): 0.8260    D(G(z)): 0.2582 / 0.0606
[2/5][1400/1583]        Loss_D: 0.5281  Loss_G: 2.7742  D(x): 0.7970    D(G(z)): 0.2275 / 0.0835
[2/5][1450/1583]        Loss_D: 0.6558  Loss_G: 1.8152  D(x): 0.6103    D(G(z)): 0.0795 / 0.2030
[2/5][1500/1583]        Loss_D: 0.9446  Loss_G: 1.1492  D(x): 0.4593    D(G(z)): 0.0356 / 0.3947
[2/5][1550/1583]        Loss_D: 0.9269  Loss_G: 0.7383  D(x): 0.5226    D(G(z)): 0.1333 / 0.5205
[3/5][0/1583]   Loss_D: 0.4855  Loss_G: 2.1548  D(x): 0.7157    D(G(z)): 0.1059 / 0.1568
[3/5][50/1583]  Loss_D: 0.7259  Loss_G: 1.1093  D(x): 0.5804    D(G(z)): 0.0797 / 0.3894
[3/5][100/1583] Loss_D: 0.7367  Loss_G: 1.0389  D(x): 0.5515    D(G(z)): 0.0405 / 0.4190
[3/5][150/1583] Loss_D: 0.5942  Loss_G: 3.4803  D(x): 0.9290    D(G(z)): 0.3709 / 0.0432
[3/5][200/1583] Loss_D: 1.3464  Loss_G: 0.6549  D(x): 0.3261    D(G(z)): 0.0242 / 0.5949
[3/5][250/1583] Loss_D: 0.5110  Loss_G: 2.2086  D(x): 0.7263    D(G(z)): 0.1327 / 0.1457
[3/5][300/1583] Loss_D: 1.4272  Loss_G: 3.3018  D(x): 0.9230    D(G(z)): 0.6654 / 0.0635
[3/5][350/1583] Loss_D: 0.6491  Loss_G: 3.0766  D(x): 0.8124    D(G(z)): 0.3127 / 0.0607
[3/5][400/1583] Loss_D: 0.5583  Loss_G: 2.9363  D(x): 0.8233    D(G(z)): 0.2759 / 0.0666
[3/5][450/1583] Loss_D: 0.9496  Loss_G: 0.6436  D(x): 0.4958    D(G(z)): 0.1367 / 0.5538
[3/5][500/1583] Loss_D: 0.4463  Loss_G: 2.2234  D(x): 0.7776    D(G(z)): 0.1545 / 0.1371
[3/5][550/1583] Loss_D: 0.5874  Loss_G: 3.6688  D(x): 0.8478    D(G(z)): 0.2930 / 0.0348
[3/5][600/1583] Loss_D: 0.3724  Loss_G: 2.6326  D(x): 0.8673    D(G(z)): 0.1854 / 0.0891
[3/5][650/1583] Loss_D: 0.7292  Loss_G: 4.4254  D(x): 0.9081    D(G(z)): 0.4234 / 0.0200
[3/5][700/1583] Loss_D: 0.4728  Loss_G: 2.8665  D(x): 0.8189    D(G(z)): 0.2115 / 0.0774
[3/5][750/1583] Loss_D: 0.5845  Loss_G: 3.3046  D(x): 0.8977    D(G(z)): 0.3490 / 0.0463
[3/5][800/1583] Loss_D: 0.5597  Loss_G: 2.2564  D(x): 0.7088    D(G(z)): 0.1497 / 0.1300
[3/5][850/1583] Loss_D: 0.6518  Loss_G: 2.5048  D(x): 0.7195    D(G(z)): 0.2183 / 0.1053
[3/5][900/1583] Loss_D: 0.7340  Loss_G: 1.4263  D(x): 0.6285    D(G(z)): 0.1806 / 0.2818
[3/5][950/1583] Loss_D: 1.4633  Loss_G: 4.9204  D(x): 0.9792    D(G(z)): 0.7093 / 0.0143
[3/5][1000/1583]        Loss_D: 0.6643  Loss_G: 2.8332  D(x): 0.8548    D(G(z)): 0.3597 / 0.0751
[3/5][1050/1583]        Loss_D: 0.7741  Loss_G: 2.9355  D(x): 0.7281    D(G(z)): 0.3064 / 0.0712
[3/5][1100/1583]        Loss_D: 0.7279  Loss_G: 3.2299  D(x): 0.8867    D(G(z)): 0.4193 / 0.0544
[3/5][1150/1583]        Loss_D: 0.6049  Loss_G: 1.9150  D(x): 0.6917    D(G(z)): 0.1645 / 0.1912
[3/5][1200/1583]        Loss_D: 0.7431  Loss_G: 3.8188  D(x): 0.9334    D(G(z)): 0.4500 / 0.0306
[3/5][1250/1583]        Loss_D: 0.5061  Loss_G: 1.9905  D(x): 0.7393    D(G(z)): 0.1531 / 0.1653
[3/5][1300/1583]        Loss_D: 0.6979  Loss_G: 3.0183  D(x): 0.8182    D(G(z)): 0.3421 / 0.0616
[3/5][1350/1583]        Loss_D: 0.9133  Loss_G: 4.0629  D(x): 0.9198    D(G(z)): 0.5131 / 0.0261
[3/5][1400/1583]        Loss_D: 0.7075  Loss_G: 4.0061  D(x): 0.9188    D(G(z)): 0.4216 / 0.0266
[3/5][1450/1583]        Loss_D: 0.7704  Loss_G: 2.3802  D(x): 0.7555    D(G(z)): 0.3348 / 0.1114
[3/5][1500/1583]        Loss_D: 0.6055  Loss_G: 1.8402  D(x): 0.7011    D(G(z)): 0.1643 / 0.1995
[3/5][1550/1583]        Loss_D: 0.7240  Loss_G: 3.2589  D(x): 0.8747    D(G(z)): 0.4069 / 0.0528
[4/5][0/1583]   Loss_D: 0.8162  Loss_G: 2.8040  D(x): 0.8827    D(G(z)): 0.4435 / 0.0870
[4/5][50/1583]  Loss_D: 0.5859  Loss_G: 2.2796  D(x): 0.6782    D(G(z)): 0.1312 / 0.1309
[4/5][100/1583] Loss_D: 0.6655  Loss_G: 3.5365  D(x): 0.8178    D(G(z)): 0.3262 / 0.0394
[4/5][150/1583] Loss_D: 1.8662  Loss_G: 5.4950  D(x): 0.9469    D(G(z)): 0.7590 / 0.0113
[4/5][200/1583] Loss_D: 0.7060  Loss_G: 3.6253  D(x): 0.9215    D(G(z)): 0.4316 / 0.0364
[4/5][250/1583] Loss_D: 0.5589  Loss_G: 2.1394  D(x): 0.7108    D(G(z)): 0.1513 / 0.1548
[4/5][300/1583] Loss_D: 0.7278  Loss_G: 1.2391  D(x): 0.5757    D(G(z)): 0.0987 / 0.3454
[4/5][350/1583] Loss_D: 0.7597  Loss_G: 2.8481  D(x): 0.7502    D(G(z)): 0.3094 / 0.0843
[4/5][400/1583] Loss_D: 0.6167  Loss_G: 2.2143  D(x): 0.6641    D(G(z)): 0.1315 / 0.1405
[4/5][450/1583] Loss_D: 0.6234  Loss_G: 1.7961  D(x): 0.7303    D(G(z)): 0.2208 / 0.2007
[4/5][500/1583] Loss_D: 0.6098  Loss_G: 4.9416  D(x): 0.9442    D(G(z)): 0.3978 / 0.0104
[4/5][550/1583] Loss_D: 0.6570  Loss_G: 3.6935  D(x): 0.9180    D(G(z)): 0.4015 / 0.0312
[4/5][600/1583] Loss_D: 0.4195  Loss_G: 2.3446  D(x): 0.7798    D(G(z)): 0.1319 / 0.1211
[4/5][650/1583] Loss_D: 0.5291  Loss_G: 2.5303  D(x): 0.7528    D(G(z)): 0.1875 / 0.1075
[4/5][700/1583] Loss_D: 0.5187  Loss_G: 2.0350  D(x): 0.7174    D(G(z)): 0.1431 / 0.1547
[4/5][750/1583] Loss_D: 0.8208  Loss_G: 1.0780  D(x): 0.5665    D(G(z)): 0.1128 / 0.3844
[4/5][800/1583] Loss_D: 0.5223  Loss_G: 3.0140  D(x): 0.8708    D(G(z)): 0.2871 / 0.0612
[4/5][850/1583] Loss_D: 2.9431  Loss_G: 1.0175  D(x): 0.0914    D(G(z)): 0.0162 / 0.4320
[4/5][900/1583] Loss_D: 0.5456  Loss_G: 1.7923  D(x): 0.7489    D(G(z)): 0.1972 / 0.2038
[4/5][950/1583] Loss_D: 0.4718  Loss_G: 2.3825  D(x): 0.7840    D(G(z)): 0.1772 / 0.1172
[4/5][1000/1583]        Loss_D: 0.5174  Loss_G: 2.5070  D(x): 0.8367    D(G(z)): 0.2556 / 0.1074
[4/5][1050/1583]        Loss_D: 0.8214  Loss_G: 0.8055  D(x): 0.5181    D(G(z)): 0.0694 / 0.4963
[4/5][1100/1583]        Loss_D: 1.3243  Loss_G: 0.7562  D(x): 0.3284    D(G(z)): 0.0218 / 0.5165
[4/5][1150/1583]        Loss_D: 0.9334  Loss_G: 5.1260  D(x): 0.8775    D(G(z)): 0.4817 / 0.0088
[4/5][1200/1583]        Loss_D: 0.5141  Loss_G: 2.7230  D(x): 0.8067    D(G(z)): 0.2188 / 0.0872
[4/5][1250/1583]        Loss_D: 0.6007  Loss_G: 1.9893  D(x): 0.6968    D(G(z)): 0.1667 / 0.1748
[4/5][1300/1583]        Loss_D: 0.4025  Loss_G: 2.3066  D(x): 0.8101    D(G(z)): 0.1471 / 0.1412
[4/5][1350/1583]        Loss_D: 0.5979  Loss_G: 3.2825  D(x): 0.8248    D(G(z)): 0.3003 / 0.0509
[4/5][1400/1583]        Loss_D: 0.7430  Loss_G: 3.6521  D(x): 0.8888    D(G(z)): 0.4243 / 0.0339
[4/5][1450/1583]        Loss_D: 1.0814  Loss_G: 5.4255  D(x): 0.9647    D(G(z)): 0.5842 / 0.0070
[4/5][1500/1583]        Loss_D: 1.7211  Loss_G: 0.7875  D(x): 0.2588    D(G(z)): 0.0389 / 0.5159
[4/5][1550/1583]        Loss_D: 0.5871  Loss_G: 2.1340  D(x): 0.7332    D(G(z)): 0.1982 / 0.1518

```

## Results

Finally, lets check out how we did. Here, we will look at three different results. First, we will see how D and G’s losses changed during training. Second, we will visualize G’s output on the fixed_noise batch for every epoch. And third, we will look at a batch of real data next to a batch of fake data from G.

**Loss versus training iteration**

Below is a plot of D & G’s losses versus training iterations.

```py
plt.figure(figsize=(10,5))
plt.title("Generator and Discriminator Loss During Training")
plt.plot(G_losses,label="G")
plt.plot(D_losses,label="D")
plt.xlabel("iterations")
plt.ylabel("Loss")
plt.legend()
plt.show()

```

![https://pytorch.org/tutorials/_images/sphx_glr_dcgan_faces_tutorial_002.png](img/097cd68a7de6371c697afbe4230ef328.jpg)

**Visualization of G’s progression**

Remember how we saved the generator’s output on the fixed_noise batch after every epoch of training. Now, we can visualize the training progression of G with an animation. Press the play button to start the animation.

```py
#%%capture
fig = plt.figure(figsize=(8,8))
plt.axis("off")
ims = [[plt.imshow(np.transpose(i,(1,2,0)), animated=True)] for i in img_list]
ani = animation.ArtistAnimation(fig, ims, interval=1000, repeat_delay=1000, blit=True)

HTML(ani.to_jshtml())

```

![https://pytorch.org/tutorials/_images/sphx_glr_dcgan_faces_tutorial_003.png](img/2a31b55ef7bfff0c24c35bc635656078.jpg)

**Real Images vs. Fake Images**

Finally, lets take a look at some real images and fake images side by side.

```py
# Grab a batch of real images from the dataloader
real_batch = next(iter(dataloader))

# Plot the real images
plt.figure(figsize=(15,15))
plt.subplot(1,2,1)
plt.axis("off")
plt.title("Real Images")
plt.imshow(np.transpose(vutils.make_grid(real_batch[0].to(device)[:64], padding=5, normalize=True).cpu(),(1,2,0)))

# Plot the fake images from the last epoch
plt.subplot(1,2,2)
plt.axis("off")
plt.title("Fake Images")
plt.imshow(np.transpose(img_list[-1],(1,2,0)))
plt.show()

```

![https://pytorch.org/tutorials/_images/sphx_glr_dcgan_faces_tutorial_004.png](img/c0f8a413c1f6dd23bb137d8adff1adda.jpg)

## Where to Go Next

We have reached the end of our journey, but there are several places you could go from here. You could:

*   Train for longer to see how good the results get
*   Modify this model to take a different dataset and possibly change the size of the images and the model architecture
*   Check out some other cool GAN projects [here](https://github.com/nashory/gans-awesome-applications)
*   Create GANs that generate [music](https://deepmind.com/blog/wavenet-generative-model-raw-audio/)
