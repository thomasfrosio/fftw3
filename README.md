This is a CMake wrapper of the FFTW 3.3.10 library.

Changes:
- CMake targets are `FFTW3::fftw3` and `FFTW3::fftw3f`. They are built in the same build directory, and include multithreading.
- In order to not collide with the system wide FFTW3 which may be a different version (e.g. `/usr/include/fftw3.h`), we also install the header inside the `{includedir}/fftw3` directory. Doing so, projects can then do `#include <fftw3/fftw3.h>` instead of `#include <fftw3.h>`.
