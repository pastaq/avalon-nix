# Avalon: Qwen3.8-Flash-Next on one AMD GPU

[Avalon](https://github.com/freriadoc/Avalon) is a llama.cpp fork that serves
`Qwen3.8-Flash-Next-Avalon-GGUF` on a single AMD card: the full model is memory-mapped from RAM and
only a subset of its experts is resident on the GPU. This flake packages it for NixOS - the HIP
engine, the quantized weights, the MTP draft head, the image encoder, and a service unit.

Everything is pinned: the engine to the commit `ca9af54` of the fork, the weights to the Hugging Face
revision `372fe3a4`, the image encoder to `ed59f92`, each file with its own hash. Nothing is downloaded
at build time except those pinned files, and the compile itself needs no network.

## Hardware

| | Minimum |
|---|---|
| GPU | RDNA3 with 20 GB+ of VRAM, offline (upstream measures gfx1100, the RX 7900 XT/XTX), a gfx1151 APU, or RDNA4 (upstream's installer takes it, one user reports it working, unmeasured) |
| RAM | 16 GB is upstream's floor now (generation stays close, new long prompts are read again from disk); 32 GB or more is the comfortable line, 64 GB or more on a gfx1151 APU, whose GPU memory is system memory |
| Disk | ~98 GiB: the 89 GiB of weights plus the build |

`hipArchs` below decides which cards the compiled engine runs on. Upstream builds and measures
`gfx1100`/`gfx1101`/`gfx1102`; its `install.sh` also takes RDNA4 (`gfx1201`/`gfx1200`) past a warning -
a user reported 86.6 t/s on a Radeon AI PRO R9700 - and refuses APUs, while the default here adds
`gfx1151` (Ryzen AI MAX / Strix Halo, RDNA3.5), which the fork's own ROCm CI runs. Any
gfx9/gfx10/gfx11/gfx12 ISA is accepted, but anything outside the default four is unmeasured, and the
chunked prompt GDN kernel is RDNA3-only, so an RDNA4 host asks for its own ISA with
`services.avalon.hipArchs = [ "gfx1201" ];`. Below 20 GB of VRAM the 256k context with MTP may not
fit - `services.avalon.maxContext = 65536;` is upstream's suggestion.

Two things matter on a gfx1151 APU:

- batched inference can return wrong output while kernel launches are asynchronous, so set
  `services.avalon.launchBlocking = true;` (upstream's gfx1151 CI does the same). It serializes
  launches and costs throughput, which is why the module leaves it off.
- the card has no VRAM of its own: it maps system RAM through the GTT, ~50 % of RAM by default. AMD
  additionally requires kernel 6.18.4 or newer for KFD to set up its queues, and the pinned nixpkgs
  ships 6.18.59. Increase the mapped pool with
  `boot.kernelParams = [ "ttm.pages_limit=26214400" ]` (100 GiB in 4 KiB pages) if the GPU side needs
  more than that half.

The IQ3_KS quant holds a 262144-token context; above that its dequant scales overflow. The MTP draft
head proposes two tokens per step.

## Packages

```console
$ nix build .#avalon          # the engine: llama-server and its CLI tools
$ nix build .#avalon-mtp      # the shared MTP draft head, 1.4 GiB
$ nix build .#avalon-mmproj   # the shared image encoder, 907 MB
$ nix build .#avalon-model    # the quantized model, 89 GiB, with the draft head in MTP/
```

`avalon-model` is the model's 33 shards, the expert ranking (`moe-cache-ranking.bin`) and the MTP
head, linked under their upstream names so llama.cpp can resolve the split model by loading the first
shard:

```console
$ nix build .#avalon-model
$ ./result/Qwen3.8-Flash-Next-Avalon-IQ3KS-00001-of-00033.gguf   # the file passed to -m
$ ./result/MTP/                                                  # the draft head passed to -md
```

`avalon-mmproj` stays out of `avalon-model` by default - it is a separate download, and text-only hosts
have no use for it. `pkgs.avalon-model.override { mmprojPackage = pkgs.avalon-mmproj; }` links it into the
model directory as `vision/` for a host that wants one path; the service takes the projector on its own
either way.

All three weight packages carry `meta.unfree = true`: they are redistributable, not free. Building them
needs `allowUnfree`, which the flake's own package set already enables, so `nix build .#avalon-model`
works as-is. With the overlay (`overlays.default`) into your own nixpkgs, set
`nixpkgs.config.allowUnfree = true` yourself.

## Running it without NixOS

`llama-server` needs the launch flags upstream's `install.sh` writes into its launcher; the smallest
reproduction is:

```console
$ nix run .#avalon -- --help                    # the server binary on its own
$ nix shell .#avalon .#avalon-model --command llama-server \
    -m ./result/Qwen3.8-Flash-Next-Avalon-IQ3KS-00001-of-00033.gguf \
    -c 262144 -b 9536 -ub 9536 -fa on -ctk q8_0 -ctv q4_0 --jinja --parallel 1 \
    -md ./result/MTP/mtp-Qwen3.8-Flash-Next-shared-IQ3K-IQ4NL-FR65k.gguf \
    --spec-type draft-mtp --spec-draft-n-max 2 \
    -ubd 256 -ctkd q8_0 -ctvd q8_0 \
    --host 127.0.0.1 --port 8080
```

The first prompt loads the graph and fills the expert cache, which takes minutes on a small card -
`llama-server` prints nothing until it is ready. Server-side tools (`llama-bench`,
`llama-batched-bench`, `llama-fit-params`) are in the same output.

## Images

The engine builds llama.cpp's mtmd library already - `tools/server` links it, and this flake builds the
tools tree - so pictures are a weight and a flag rather than a rebuild: `-mm PROJECTOR`. Avalon's own
repository publishes no projector, so `avalon-mmproj` pins the one the whole Qwen3.8-Flash-Next line
shares, from ISTA's GSQ-RCO quantization - the file Strata serves images with. The Avalon quant keeps
ISTA's experts and requantizes only the up/gate tensors of the language model, so the tower and its merger
still fit it: the projector emits 2560-wide rows, which is the embedding length of the `qwen4exp`
architecture the model declares, and the model's own chat template renders `image_url` parts into
`vision_start`/`vision_end`.

```console
$ nix build .#avalon-mmproj   # 449M-parameter vision tower, BF16, 907 MB
```

Without NixOS, add the flag to the line from the section above:

```console
$ MM=$(nix build --no-link --print-out-paths .#avalon-mmproj)
$ nix shell .#avalon .#avalon-model --command llama-server \
    -m ./result/Qwen3.8-Flash-Next-Avalon-IQ3KS-00001-of-00033.gguf \
    -mm "$MM/mmproj-Qwen3.8-Flash-Next-BF16.gguf" \
    -c 262144 -b 9536 -ub 9536 -fa on -ctk q8_0 -ctv q4_0 --jinja --parallel 1 \
    -md ./result/MTP/mtp-Qwen3.8-Flash-Next-shared-IQ3K-IQ4NL-FR65k.gguf \
    --spec-type draft-mtp --spec-draft-n-max 2 \
    -ubd 256 -ctkd q8_0 -ctvd q8_0 \
    --host 127.0.0.1 --port 8080
```

```console
$ curl -s http://127.0.0.1:8080/v1/chat/completions -H 'Content-Type: application/json' \
    -d "{\"messages\":[{\"role\":\"user\",\"content\":[" \
       "{\"type\":\"image_url\",\"image_url\":{\"url\":\"data:image/png;base64,$(base64 -w0 pic.png)\"}}," \
       "{\"type\":\"text\",\"text\":\"What is in this picture?\"}]}]}"
```

On NixOS:

```nix
services.avalon.vision.enable = true;   # adds -mm, and the projector, to the unit
```

Options: `vision.enable`, `vision.package`, `vision.file`, `vision.maxTokens`, `vision.offload`.

Four things to know:

- The tower is BF16, so encoding on the card wants ~0.9 GiB of VRAM on top of the 19.7 GiB the 256k
  context already holds with MTP and the expert cache. On a 20 GB card use `vision.offload = false` - the
  encoder then runs on the CPU, slower per picture - or `services.avalon.maxContext = 65536;`.
- The encoder works at 768x768 with 16-pixel patches and a 2x2 merge, so a full-resolution picture is 576
  tokens. `vision.maxTokens` caps that (`--image-max-tokens`); left null, the file's own value is used.
- Avalon has never measured MTP with pictures, and nothing in the server forbids the combination. If a
  multimodal answer comes back wrong, drop the drafter: `services.avalon.extraArgs = [ "--spec-type"
  "none" ];`. `extraArgs` is appended, so it overrides the module's `--spec-type draft-mtp`.
- The web UI is not built (`LLAMA_BUILD_UI` is off), so there is no drag-and-drop box; pictures arrive
  through the OpenAI-compatible endpoint.

webp pictures and video are decoded by running `ffmpeg`, which the unit does not carry: png and jpeg are
decoded in process and need nothing. To accept webp, put it on the unit's `PATH` - `env` takes precedence
over the module's own environment:

```nix
services.avalon.env.PATH = "${pkgs.ffmpeg}/bin:/run/current-system/sw/bin";
```

## NixOS

```nix
{ inputs, pkgs, ... }:
{
  nixpkgs.overlays = [ inputs.avalon.overlays.default ];
  nixpkgs.config.allowUnfree = true;

  services.avalon = {
    enable = true;
    # The card's own ISA; the default builds the three desktop RDNA3 ISAs upstream tests plus gfx1151.
    hipArchs = [ "gfx1151" ];
    launchBlocking = true;  # correct output on gfx1151; leave off on a discrete card
    openFirewall = false;   # the server binds 127.0.0.1:8080
  };
}
```

The unit runs `llama-server` as the `avalon` user from `/var/lib/avalon`, with `render` and `video`
groups, `/dev/kfd` and `/dev/dri` allowed, `LimitMEMLOCK=infinity` (ROCm pins the resident experts),
and no start timeout. Its command line is the launcher flags above; `extraArgs` appends to it and
`env` covers the engine's tuning switches, e.g.:

```nix
services.avalon = {
  extraArgs = [ "--moe-cache-cpu-overlap" "2" ];   # a share of the cached experts computed on the CPU
  env = {
    AMD_P2U_WAIT_ENABLE = "1";
    HIPBLASLT_HEURISTICS_CACHE_CAPACITY = "3000";
    LLAMA_QSA_TILES = "0";   # back to the per-slice QSA attention of the older engine
  };
};
```

Options: `package`, `modelPackage`, `hipArchs`, `launchBlocking`, `maxContext`, `batchSize`,
`unifiedBatchSize`, `draftBatchSize`, `draftMax`, `cacheQuantK`, `cacheQuantV`, `draftCacheQuant`,
`modelFile`, `mtpFile`, the `vision.*` set described under [Images](#images), `host`, `port`, `stateDir`,
`user`, `group`, `extraArgs`, `env`, `openFirewall`. Changing `hipArchs` rebuilds the engine; turning
`vision.enable` on does not.

To check the server answers:

```console
$ curl -s http://127.0.0.1:8080/v1/models | head
```

## Development

```console
$ nix flake show     # packages.avalon / avalon-model / avalon-mtp / default, nixosModules.avalon
$ nix flake check    # evaluates the outputs; it does not build the packages
$ nix build --impure -f tests/module.nix --no-link   # the module into a system toplevel
```

`tests/module.nix` evaluates the service into a bootable NixOS system, which is how to check the unit's
dir, groups, limits and command line; realising it fetches the pinned weights.

The engine builds with ROCm from nixpkgs (`rocmPackages.clang` and `rocmPackages.rocblas`); this tree
pins nixpkgs-unstable, which ships ROCm 7.2.4 - the version upstream tested. There is no prebuilt
cache: `nix build .#avalon` compiles HIP kernels for each of the chosen ISAs, roughly twenty minutes
for the default four on a 16-core machine.

To move the engine to a newer Avalon commit, change `rev` in `pkgs/by-name/av/avalon/package.nix` and
set `version` to what that commit's `CMakeLists.txt` reports for `LLAMA_VERSION` (`0.8.0-dev` at the
commit pinned here); the build reports the new SRI hash on the first attempt, or
`nix-prefetch-url --unpack <archive url>` gives it without a build. Read the commits since the pin for
changes to the build and to the launcher's flags before bumping. To move the weights, change
`revision` in `pkgs/by-name/av/avalon-model/package.nix` and
re-pin the files whose hashes the build reports. To move the image encoder, change `revision` in
`pkgs/by-name/av/avalon-mmproj/package.nix`; its file name is the default of
`services.avalon.vision.file`, so a rename means changing both.

## License

MIT for the engine (as upstream); Apache 2.0 for the quantized weights. The image encoder carries the Qwen
community license its own file declares. The weights are redistributable but not free, so they are marked
`unfree` in the package metadata.
