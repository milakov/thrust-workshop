# ########################
# GTC 2014 Thrust Workshop
# ########################

NVCC_OPTIONS=-gencode=arch=compute_20,code=sm_20 -gencode=arch=compute_30,code=\"compute_30,sm_30\"
all: exercise spoilers cpu_performance gpu_performance

exercise: exercise.cu
        nvcc $(NVCC_OPTIONS) -o exercise.o -c -Xcompiler -O3 -I . exercise.cu
        gcc -o exercise exercise.o -L/usr/local/cuda/lib64 -lstdc++ -lcudart

spoilers: spoilers.cu
        nvcc $(NVCC_OPTIONS) -o spoilers.o -c -Xcompiler -O3 -I . spoilers.cu
        gcc -o spoilers spoilers.o -L/usr/local/cuda/lib64 -lstdc++ -lcudart

cpu_performance: performance.cu
        nvcc $(NVCC_OPTIONS) -o cpu_performance.o -c -DTHRUST_DEVICE_SYSTEM=THRUST_DEVICE_SYSTEM_CPP -Xcompiler -O3 -I . performance.cu
        gcc -o cpu_performance cpu_performance.o -L/usr/local/cuda/lib64 -lstdc++ -lcudart

gpu_performance: performance.cu
        nvcc $(NVCC_OPTIONS) -o gpu_performance.o -c -DTHRUST_DEVICE_SYSTEM=THRUST_DEVICE_SYSTEM_CUDA -Xcompiler -O3 -I . performance.cu
        gcc -o gpu_performance gpu_performance.o -L/usr/local/cuda/lib64 -lstdc++ -lcudart

clean:
        rm -f exercise spoilers cpu_performance gpu_performance exercise.o spoilers.o cpu_performance.o gpu_performance.o
