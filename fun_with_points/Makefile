# ########################
# GTC 2013 Thrust Workshop
# ########################
#
# IMPORTANT NOTE
# ##############
# This example uses a Makefile since the target environment (the Amazon Machine
# Image) is constrained. For more generality, you could use scons, for which the
# configuration is provided in the GitHub hosted repository:
#
#     https://github.com/tmjbradley/thrust-workshop
# 

all: exercise spoilers cpu_performance gpu_performance

exercise: exercise.cu
	nvcc -o exercise.o -c -Xcompiler -O3 -I . exercise.cu
	gcc -o exercise exercise.o -L/usr/local/cuda/lib64 -lstdc++ -lcudart

spoilers: spoilers.cu
	nvcc -o spoilers.o -c -Xcompiler -O3 -I . spoilers.cu
	gcc -o spoilers spoilers.o -L/usr/local/cuda/lib64 -lstdc++ -lcudart

cpu_performance: performance.cu
	nvcc -o cpu_performance.o -c -DTHRUST_DEVICE_SYSTEM=THRUST_DEVICE_SYSTEM_CPP -Xcompiler -O3 -I . performance.cu
	gcc -o cpu_performance cpu_performance.o -L/usr/local/cuda/lib64 -lstdc++ -lcudart

gpu_performance: performance.cu
	nvcc -o gpu_performance.o -c -DTHRUST_DEVICE_SYSTEM=THRUST_DEVICE_SYSTEM_CUDA -Xcompiler -O3 -I . performance.cu
	gcc -o gpu_performance gpu_performance.o -L/usr/local/cuda/lib64 -lstdc++ -lcudart

clean:
	rm -f exercise spoilers cpu_performance gpu_performance exercise.o spoilers.o cpu_performance.o gpu_performance.o

