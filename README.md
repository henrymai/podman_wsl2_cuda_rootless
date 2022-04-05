# Setup podman for WSL2 with cuda support (rootless)

## Upgrade Ubuntu 20.04 LTS to 20.10:

```
sudo apt-get remove snapd
sudo apt-get update
sudo apt-get upgrade
sudo do-release-upgrade
```


## Install podman:

```
sudo apt-get update
sudo apt-get install podman
```


## Add docker.io registry:

```
echo 'unqualified-search-registries = ["docker.io"]' | sudo tee /etc/containers/registries.conf
```


## Install nvidia-container-toolkit:

```
distribution=ubuntu20.04
curl -s -L https://nvidia.github.io/libnvidia-container/gpgkey | sudo apt-key add - && curl -s -L https://nvidia.github.io/libnvidia-container/$distribution/libnvidia-container.list | sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list

sudo apt-get update
sudo apt-get install nvidia-container-toolkit
```


## Rootless configuration for nvidia container runtime:

Modify: `/etc/nvidia-container-runtime/config.toml`

```
[nvidia-container-cli]
#no-cgroups = false
no-cgroups = true

[nvidia-container-runtime]
#debug = "/var/log/nvidia-container-runtime.log"
debug = "~/.local/nvidia-container-runtime.log"
```


## Setup missing hook for nvidia container runtime:

```
sudo mkdir -p /usr/share/containers/oci/hooks.d/

cat << EOF | sudo tee /usr/share/containers/oci/hooks.d/oci-nvidia-hook.json
{
    "version": "1.0.0",
    "hook": {
        "path": "/usr/bin/nvidia-container-toolkit",
        "args": ["nvidia-container-toolkit", "prestart"],
        "env": [
            "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
        ]
    },
    "when": {
        "always": true,
        "commands": [".*"]
    },
    "stages": ["prestart"]
}
EOF
```

## Increase memlock and stack ulimits
* necessary otherwise any reasonable sized training run will hit these limits immediately

Modify: `/etc/security/limits.conf`

Assuming your username is `someuser`:

```
someuser soft memlock unlimited
someuser hard memlock unlimited
someuser soft stack 65536
someuser hard stack 65536
```

Note that before each time you run a podman container, you will have to run: `su someuser` to update the ulimits for your bash session or prefix your command with `su someuser -c`


## Test that it works:

```
su someuser -c "podman run --rm nvidia/cuda:11.0-base nvidia-smi; cat /proc/self/limits"
```

### Output:

```
Resolving "nvidia/cuda" using unqualified-search registries (/etc/containers/registries.conf)
Trying to pull docker.io/nvidia/cuda:11.0-base...
Getting image source signatures
Copying blob 3642f1a6dfb3 done
Copying blob b66c17bbf772 done
Copying blob e5ce55b8b4b9 done
Copying blob 54ee1f796a1e done
Copying blob 46d371e02073 done
Copying blob f7bfea53ad12 done
Copying blob 155bc0332b0a done
Copying config 2ec708416b done
Writing manifest to image destination
Storing signatures
Tue Apr  5 21:44:04 2022
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 510.47.03    Driver Version: 511.65       CUDA Version: 11.6     |
|-------------------------------+----------------------+----------------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
|                               |                      |               MIG M. |
|===============================+======================+======================|
|   0  NVIDIA TITAN RTX    On   | 00000000:01:00.0  On |                  N/A |
| 40%   35C    P8    19W / 280W |   1356MiB / 24576MiB |      1%      Default |
|                               |                      |                  N/A |
+-------------------------------+----------------------+----------------------+

+-----------------------------------------------------------------------------+
| Processes:                                                                  |
|  GPU   GI   CI        PID   Type   Process name                  GPU Memory |
|        ID   ID                                                   Usage      |
|=============================================================================|
|    0   N/A  N/A     10269      G   /Xwayland                       N/A      |
+-----------------------------------------------------------------------------+
Limit                     Soft Limit           Hard Limit           Units
Max cpu time              unlimited            unlimited            seconds
Max file size             unlimited            unlimited            bytes
Max data size             unlimited            unlimited            bytes
Max stack size            67108864             67108864             bytes
Max core file size        0                    unlimited            bytes
Max resident set          unlimited            unlimited            bytes
Max processes             63512                63512                processes
Max open files            1024                 1048576              files
Max locked memory         unlimited            unlimited            bytes
Max address space         unlimited            unlimited            bytes
Max file locks            unlimited            unlimited            locks
Max pending signals       63512                63512                signals
Max msgqueue size         819200               819200               bytes
Max nice priority         0                    0
Max realtime priority     0                    0
Max realtime timeout      unlimited            unlimited            us
```
