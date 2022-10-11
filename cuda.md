# Cuda

- [Cuda](#cuda)
  - [Motivation](#motivation)
    - [Architecture du CPU](#architecture-du-cpu)
    - [Architecture du GPU](#architecture-du-gpu)
  - [Execution d'un programme CUDA](#execution-dun-programme-cuda)
    - [Étapes](#étapes)
  - [Librairie](#librairie)
    - [Fonctions](#fonctions)
    - [Configuration du parallélisme](#configuration-du-parallélisme)
      - [Déclaration de fonction](#déclaration-de-fonction)
      - [Appel de fonction](#appel-de-fonction)
      - [Exemple](#exemple)
    - [Parallélisme à 3 dimensions](#parallélisme-à-3-dimensions)

## Motivation

Les GPUs (Graphic Processing Units) sont conçus pour du rendu en temps réel (jeux vidéos). Les exigences de traitements sont croissantes.
On utilise donc les ressources du GPU pour les calculs => GPGPU General-Purpose GPU computing.

### Architecture du CPU

Extraction du parallélisme dans un thread (exécution superscalaire, sans ordre)

- 1 Control
- 1 Cache
- N ALU (Arithmetic Logic Unit) => N coeurs
- DRAM

### Architecture du GPU

Exécution SIMT (single instruction-multiple threads)

- N Control
- N Cache
- N*15 (?) ALU
- DRAM

## Execution d'un programme CUDA

Le CPU initialise l'execution des kernels sur le GPU. Le GPU crée un grand nombre de threads qui exécutent le code du kernel.

### Étapes

Code du CPU
Code du GPU (kernel)

1. Allocation mémoire GPU (`cudaMalloc`)
2. Transfert des données CPU vers GPU, host vers device(`cudaMemcpy`)
3. Exécution du kernel
4. Transfert des données GPU vers CPU, device vers host (`cudaMemcpy`)

## Librairie

### Fonctions

- `cudaError_t cudaMalloc (void **devPtr, size_t size)`.
  Alloue la mémoire sur le device.
  - `devPtr`: Pointeur vers la mémoire allouée du device
  - `size`: Taille de la mémoire allouée en octet
- `cudaError_t cudaMemcpy (void *dst, const void *src, size_t count, cudaMemcpyKind kind)`. Copie les données entre host et device.
  - `dst`: Adresse mémoire de la destination
  - `src`: Taille de la mémoire allouée en bytes
  - `count`: Taille en octet à copier
  - `kind`: `cudaMemcpyHostToDevice` ou `cudaMemcpyDevicetoHost`
- `cudaError_t cudeFree (void *devPtr)`.
  Libère la mémoire sur le device.
  - `devPtr`: Pointeur vers la mémoire allouée du device à libérer

### Configuration du parallélisme

#### Déclaration de fonction

`__global__ void f()`: indique que la fonction est un kernel.

- Exécutée par le GPU
- Appellée par le CPU
  
`__device__ float g()`

- Exécutée par le GPU
- Appellée par le GPU

`__host__ int h()` (defaut)

- Exécutée par le CPU
- Appellée par le CPU

#### Appel de fonction

`f<<<x, y>>>(d_out, d_in)`

- `x`: nombre de blocs
- `y`: nombre de threads par bloc
- `x, y`: nombre de threads

`idx` and `f`: variables privées des threads

`threadIdx.x`: identificateur du thread

`blockIdx.x`: identificateur du bloc

`blockDim.x`: dimension du bloc ou nombre de threads dans un bloc

`gridDim.x`: nombre de blocs

Exemple de calcul d'index:

- N = 262144: Nombre de threads voulu
- y = 256
- x = N / y = 1024
  
bloc 0
|0|1|2|...|255|
|-|-|-|-|-|

bloc 1
|0|1|2|...|255|
|-|-|-|-|-|

...bloc 1023
|0|1|2|...|255|
|-|-|-|-|-|

Index du thread 2 du bloc 1: `blockIdx.x (1) * blockDim.x (256) + threadIdx.x (2) = 258`

#### Exemple

    void rgb2grey(uchar4 *hcimg, uchar *hgimg, int isize) 
    {
      int bsize = 256;
      int gsize = ((isize + bsize -1) /bsize);
      int numc = isize*sizeof(uchar4);
      int numg = isize*sizeof(unsigned char);
      uchar4* dcimg;
      unsigned char *dgimg;
      cudaMalloc((void **)&dcimg, numc);
      cudaMemcpy(dcimg, hcimg, numc, cudaMemcpyHostToDevice);
      cudaMalloc((void **)&gdimg, numg) ;
      color2grey<<< gsize, bsize >>>(dcimg , dgimg , isize);
      cudaMemcpy(hgimg, dgimg, numg, cudaMemcpyDeviceToHost);
      cudaFree(dcimg) ; cudaFree(dgimg) ;
    }

    __global__
    void color2grey(uchar4 *dcimg, uchar *dgimg, int isize)
    {
      int index = blockIdx.x * blockDim.x + threadIdx.x;
      uchar4 p;
      if (index < isize)
      {
        p = dcimg[index];
        dgimg[index] = (299*p.x + 587*p.y + 114*p.z) / 1000;
      }
    }

### Parallélisme à 3 dimensions