# Putting InstructLab in a Container AND making it go fast

Containerization of `ilab` allows for portability and ease of setup. With this,
users can now run lab on OpenShift to test the speed of `ilab model train` and `generate`
using dedicated GPUs. This guide shows you how to put the `ilab` CLI, all of its
dependencies, and your GPU into a container for an isolated and easily reproducible
experience.

The rest of the document and the `Makefile` assume that you are running a
Fedora-like Linux distribution with Podman and SELinux. If you are running
another Linux distribution with Docker, then run make with `CENGINE=docker`,
substitute `podman` with `docker`, and remove the `:z` suffix from volume
mounts.

## Build the InstructLab container image for CUDA

```shell
1. podman build -f <Containerfile_Path>
2. curl -s -L https://nvidia.github.io/libnvidia-container/stable/rpm/nvidia-container-toolkit.repo |   sudo tee /etc/yum.repos.d/nvidia-container-toolkit.repo
3. sudo yum-config-manager --enable nvidia-container-toolkit-experimental
4. sudo dnf install -y nvidia-container-toolkit
5. sudo nvidia-ctk cdi generate --output=/etc/cdi/nvidia.yaml
6. nvidia-ctk cdi list
    Example output: 
    INFO[0000] Found 2 CDI devices
    nvidia.com/gpu=0
    nvidia.com/gpu=all
7. podman run --device nvidia.com/gpu=0  --security-opt=label=disable -it <IMAGE_ID>
```

Then, you can verify that NVIDIA container toolkit can see your GPUs.

```shell
nvidia-ctk cdi list
```

Example output:

```shell
INFO[0000] Found 2 CDI devices
nvidia.com/gpu=0
nvidia.com/gpu=all
```

Generate the CDI configuration for the NVIDIA devices:

```shell
sudo nvidia-ctk cdi generate --output=/etc/cdi/nvidia.yaml
```

Finally, allow non-privileged containers to access the NVIDIA devices:

```shell
sudo setsebool -P container_use_devices on
```

### Run the CUDA-accelerated InstructLab container

When running our model, we want to create some paths that will be mounted in
the container to provide data persistence. The volume mount will contain
InstructLab's configuration, models, taxonomy, and Hugging Face download cache.

```shell
mkdir -p $HOME/.config/instructlab
```

Then, we can run the container, mounting the above folder and using the first
NVIDIA GPU.

```shell
podman run --rm -it --device nvidia.com/gpu=0 --volume $HOME/.config/instructlab:/opt/app-root/src:z localhost/instructlab:cuda
```

The above command will give you an interactive shell inside the container.
Let's initialize our configuration, download the model and start the chatbot.

```shell
(app-root) ilab init
(app-root) ilab download
(app-root) ilab chat
```

## Containerfile design

The container files use a multi-stage builder approach to keep the final
image small. There are typically several stages:

- The `runtime` stage contains system packages needed to run InstructLab.
- The `builder` stage has build tools and development files to build and
  compile additional packages.
- The `final` stage assembles `runtime` and InstructLab's virtual environment.

The container mimics the layout of the `ubi9/python-311` container with a
virtual environment in `/opt/app-root`. The working directory and user's home
is `/opt/app-root/src`.

`pip install` and `dnf install` use a cache mount to cache downloads. This
speeds up rebuilds and shares common packages between builds of multiple
containers.

Pip is configured to not compile Python code to byte code (`PIP_NO_COMPILE`)
to reduce the size of the image.