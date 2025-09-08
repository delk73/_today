You’re on **470.xx** (legacy) with a **GTX 760M (Kepler, sm\_30)**. That dictates your CUDA path:

* **Do NOT install `cuda` from Arch repos** (it’s 12.x → requires newer drivers and drops Kepler).
* Use **CUDA 10.2** from AUR (last toolchain that supports **sm\_30**). Your 470 driver will run 10.2-built binaries fine.

### Install (Arch/Omarchy, with yay)

```bash
# Build tools (for compiling samples)
sudo pacman -S --needed base-devel

# CUDA 10.2 from AUR (name may be cuda-10.2 or cuda-10.2.89 on AUR)
yay -S cuda-10.2 || yay -S cuda-10.2.89
```

### Set env for this session (AUR packages usually install under /opt)

```bash
# Most AUR builds place 10.2 at /opt/cuda-10.2
export CUDA_HOME=/opt/cuda-10.2
export PATH="$CUDA_HOME/bin:$PATH"
export LD_LIBRARY_PATH="$CUDA_HOME/lib64:$LD_LIBRARY_PATH"

# sanity
nvcc --version
```

### Build & run the stock CUDA samples

```bash
# copy samples to your home so you don't build in /opt
cp -r "$CUDA_HOME/samples" ~/cuda-samples-10.2
cd ~/cuda-samples-10.2

# 1) Device properties
cd 1_Utilities/deviceQuery
make -j"$(nproc)"
./deviceQuery     # expect "Result = PASS" and compute capability 3.0

# 2) Memory bandwidth
cd ../bandwidthTest
make -j"$(nproc)"
./bandwidthTest   # prints H2D/D2H/Device BW, expect PASS
```

### Watch the GPU while you test

```bash
watch -n1 nvidia-smi
```

You should leave `P8` (idle), see non-zero GPU-Util, and some memory usage.

### Common gotchas (and exact fixes)

* **You accidentally installed `cuda` 12.x**
  Symptoms: `nvcc fatal : Unsupported gpu architecture 'compute_30'` or runtime says driver too old.
  Fix:

  ```bash
  sudo pacman -Rns cuda
  yay -S cuda-10.2 || yay -S cuda-10.2.89
  ```
* **Samples can’t find CUDA libs at runtime**
  Fix: ensure `LD_LIBRARY_PATH` includes `$CUDA_HOME/lib64` (see env block above).
* **DKMS/driver mismatch after a kernel bump**
  You’re on **linux-lts + nvidia-470xx** — keep it that way. If modules ever fail to load:

  ```bash
  sudo dkms autoinstall
  sudo mkinitcpio -P
  ```

### Optional OpenCL sanity (if you care)

```bash
yay -S opencl-nvidia-470xx clinfo
clinfo | head
```

If any of that fails or package names differ on your AUR mirror, run:

```bash
yay -Ss '^cuda($|.*10\.2)'
```

and pick the CUDA **10.2** package it lists.
