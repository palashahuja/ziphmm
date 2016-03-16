

# Background #
Hidden Markov models are widely used for genome analysis as they
combine ease of modelling with efficient analysis
algorithms. Calculating the likelihood of a model using the forward
algorithm has worst case time complexity linear in the length of the
sequence and quadratic in the number of states in the model. For
genome analysis, however, the length runs to millions or billions of
observations, and when maximising the likelihood hundreds of
evaluations are often needed. A time efficient forward algorithm is
therefore a key ingredient in an efficient hidden Markov model
library.

# Library #
We have built a software library for efficiently computing the
likelihood of a hidden Markov model. The library exploits commonly
occurring substrings in the input to reuse computations and to
parallelise the forward algorithm on multi-core machines. In a
pre-processing step our library identifies common substrings and
builds a structure over the computations in the forward algorithm
which can be reused. This analysis can be saved between uses of the
library and is independent of concrete hidden Markov models so one
preprocessing can be used to run a number of different models.

Using the library, we achieve up to 78 times shorter wall-clock
time for realistic whole-genome analyses with a real and reasonably
complex hidden Markov model. In one particular case the analysis was
performed in less than 8 minutes compared to 9.6 hours for the
previously fastest library.

# Using the library #

## Build procedure ##

To build and install the library, unzip the directory and execute the
following commands in a terminal:

```
$ cd <path to library>/zipHMM-0.0.1/
zipHMM-1.0.0 $ cmake .
zipHMM-1.0.0 $ make
zipHMM-1.0.0 $ bin/calibrate
zipHMM-1.0.0 $ make test
zipHMM-1.0.0 $ make install
```

To build in OS X, the Accelerate framework is required (see
https://developer.apple.com/performance/accelerateframework.html). This
is included in the developer tools installed with XCode (see
https://developer.apple.com/xcode/)

To build in Linux CMake must be able to find an installation of a BLAS
implementation. For now the CMake script is set up to use Atlas and to
look for it at /com/extra/ATLAS/3.9.84. This will most likely not work
on your machine. You may therefore have to change line 11 in
`zipHMM/CmakeLists.txt`:

```
  set(ATLAS_ROOT "/com/extra/ATLAS/3.9.84")
```

If you are using a different implementation of BLAS than Atlas you
will have to do a few extra simple changes in zipHMM/CMakelists.txt -
look at line 12, 13, 32, 56, 76, 91, 106, 121, 159 and 192.

`bin/calibrate` finds the optimal number of threads to use in the
parallelized algorithm and saves the number in a file (default is
`~/.ziphmm.devices`).

## Getting started ##

Have a look at `zipHMM/cpp_example.cpp` and `zipHMM/python_example.cpp`
and try running the following commands from the root directory.

```
zipHMM-1.0.0 $ bin/cpp_example
	
zipHMM-1.0.0 $ cd zipHMM/
zipHMM $ python python_example.py
```

## Using the C++ library ##

The main class in the library is Forwarder (`forwarder.hpp` and
`forwarder.cpp`). Objects of this class represents an observed sequence
that have been preprocessed such that the likelihood of the sequence
can be obtained from a given HMM (pi, A, B) very fast. To build a new
forwarder object just call the empty constructor:

```
Forwarder();
```

and to read in a sequence call one of the three `read_seq` methods:

```
void Forwarder::read_seq(const std::string &seq_filename, const size_t alphabet_size, 
                         std::vector<size_t> nStatesSave, const size_t min_no_eval = 1);
void Forwarder::read_seq(const std::string &seq_filename, const size_t alphabet_size, 
     			 const size_t no_states, const size_t min_no_eval);
void Forwarder::read_seq(const std::string &seq_filename, const size_t alphabet_size, 
                         const size_t min_no_eval = 1)
```

Here `seq_filename` is the filename of the file containing the observed
sequence, `alphabet_size` is the size of the alphabet used in the
observed sequence, `nStatesSave` is a vector indicating the sizes of the
HMMs the user intends to run the `Forwarder` object on, and `min_no_eval`
is a guess of the number of times, the preprocessing will be reused
(if unsure about this then leave it out and use the default value). If
`nStatesSave` contains the vector (2, 4, 8), data structures obtaining
the fastest evaluation of the forward algorithm for each of the HMM
state space sizes 2, 4, and 8 will be build. If `nStatesSave` is left
empty a single data structure obtaining a very fast evaluation of the
forward algorithm for all HMM state space sizes will be saved.

The second constructor serves as a convenient way to call the first
constructor with only one HMM size in `nStatesSave`.
The third constructor serves as a convenient way to call the first
constructor with an empty `nStates2save` vector.

After building a `Forwarder` object, it can be saved to disk using the method

```
void write_to_directory(const std::string &directory) const;
```

Here `directory` should contain the path of the intended location of the datastructure.

To read a previously saved datastructure, one of the following two methods
can be used:

```
void Forwarder::read_from_directory(const std::string &dirname);
void Forwarder::read_from_directory(const std::string &directory, const size_t no_states);
```

Using the first one, the entire data structure is being rebuilt. Using
the second one only the data structure matching `no_states` is being
rebuild. This will be faster in many cases. If you did not save the
data structure for the size of your HMM, then use the first
constructor. The forward algorithm will figure out which of the saved
data structures is most optimal for your HMM.

Finally, to get the loglikelihood of the observed sequence in a
specific model, one of the following methods are used:

```
double Forwarder::forward(const Matrix &pi, const Matrix &A, const Matrix &B) const;
double Forwarder::pthread_forward(const Matrix &pi, const Matrix &A, const Matrix &B, 
                                  const std::string &device_filename = DEFAULT_DEVICE_FILENAME) const;
```

The second method is a parallelized version of the forward algorithm,
whereas the first one is single-threaded. `pi`, `A` and `B` specifies the
HMM parameters. They can either be read from a file or build in C++ as
described below in the section 'Encoding HMMs'. The parallelized version
takes an additional filename as parameter. This filename should be the
path to the file created by the calibrate program, which finds the
optimal number of threads to use in the parallelized forward
algorithm. The default filename is `~/.ziphmm.devices`. If you did not
move the file, then leave the `device_filename` parameter out.

See `zipHMM/cpp_example.cpp` for a simple example.

## Using the Python library ##

To use the Python library in another project, run `make install` as described above or copy `zipHMM/pyZipHMM.py`
and `zipHMM/libpyZipHMM.so` to the root of your project folder after
building the library and import `pyZipHMM` in your script. See `zipHMM/python_example.py`
and `zipHMM/python_test.py` for details on how to use the library.

A Forwarder object can be constructed from an observed sequence in the
following ways:

```
from pyZipHMM import *
f = Forwarder.fromSequence(seqFilename = "example.seq", alphabetSize = 3, minNoEvals = 500)
```

To save the datastructure to disk do as follows:

```
f.writeToDirectory("example_preprocessed")
```

To read a previously saved datastructure from disk use either of the
two methods:

```
f2 = Forwarder.fromDirectory(directory = "../example_out")
f2 = Forwarder.fromDirectory(directory = "../example_out", nStates = 3)
```

Finally, to evaluate the loglikelihood of the sequence in a given model
(matrices `pi`, `A` and `B`) use either of

```
loglikelihood = f.forward(pi, A, B)
loglikelihood = f.ptforward(pi, A, B)
```

where the second method is parallelized. The three matrices `pi`, `A` and `B`
can be read from a file or build in Python as described below.

See `zipHMM/python_example.py` for an example.

## Encoding HMMs ##

An HMM consists of three matrices:

> - pi, containing initial state probabilities: pi\_i is the probability of the model starting in state i.

> - A, containing transition probabilities: A_{i,j} is the probability of the transition from state i to state j._

> - B, containing emission probabilities: B_{i,o} is the probability of state i emitting symbol o._

These three matrices can either be build in the code (in C++ or
Python) or they can be encoded in a text file. The format of the text
file is as follows:

```
###### HMM example ######

no_states
3
alphabet_size
4
pi
0.1
0.2
0.7
A
0.1 0.2 0.7
0.3 0.4 0.3
0.5 0.5 0.0
B
0.1 0.2 0.3 0.4
0.2 0.3 0.4 0.1
0.3 0.4 0.1 0.2

#####################
```
To read and write HMMs from and to files in C++, use the methods

```
void read_HMM(Matrix &resultInitProbs, Matrix &resultTransProbs, Matrix &resultEmProbs, const std::string &filename);
void write_HMM(const Matrix &initProbs, const Matrix &transProbs, const Matrix &emProbs, const std::string &filename);
```

To read and write HMMs from and to files in Python, use the functions

```
readHMM(filename) -> (pi, A, B)
writeHMM(pi, A, B, filename) -> None
```

To build a matrix in C++ do as illustrated in the following example:

```
###### C++ example ######

#include "matrix.hpp"

size_t nRows = 3;
size_t nCols = 4;
zipHMM::Matrix m(3,4);
m(0,0) = 0.1;
m(0,1) = 0.2;
...
m(2,3) = 0.2;

#####################
```

To build a matrix in Python do as illustrated here:

```
## Python example ####

import pyZipHMM

nRows = 3
nCols = 4
m = pyZipHMM.Matrix(nRows, nCols)
m[0,0] = 0.1
m[0,1] = 0.2
...
m[2,3] = 0.2

#####################
```

## Encoding sequences ##

The alphabet of observables are encoded using integers. Thus if the
size of the alphabet is M, the observables are encoded using 0, 1, 2,
..., M - 1. A sequence of observables is encoded in a text file with
the single observations separated by whitespace. See `example.seq`
for an example.


## Executables ##

### calibrate ###

Usage:
```
bin/calibrate
```

Finds the optimal number of threads to use in the parallelized version
of the forward algorithm.

### build\_forwarder ###

Usage:
```
bin/build_forwarder -s <sequence filename> -M <alphabet size> -o <output directory> [-N <number of states>]*
```
Builds a Forwarder object from the sequence in the file specified in
<sequence filename> and writes it to the directory specified in
<output directory>. <alphabet size> should be the size of the alphabet
used in the observed sequence, and the file specified in <sequence
filename> should contain a single line containing white space
separated integers between 0 and <alphabet size> - 1. The list of HMM
sizes to generate the data structure for can be specified using the -N parameter.

Examples:
```
bin/build_forwarder -s example.seq -M 3 -o example_out
bin/build_forwarder -s example.seq -M 3 -o example_out -N 2
bin/build_forwarder -s example.seq -M 3 -o example_out -N 2 -N 4 -N 8 -N 16
```

### forward ###

Usage:
```
bin/forward (-s <sequence filename> -m <HMM filename> [-e number of expected forward calls] [-o <output directory>] ) | (-d <preprocessing directory> -m <HMM filename>) [-p]
```

Runs the forward algorithm and outputs the loglikelhood. This
executable can be called in two different ways:

```
bin/forward -s example.seq -m example.hmm -e 500 -o example_out
bin/forward -d example_out/ -m example.hmm
```

In the first example the loglikelihood is evaluated based on the
observed sequence in example.seq and the HMM specified in
`example.hmm`. In the second example the loglikelihood is evaluated
based on the previously saved data structure in `example_out/` and the
HMM specified in `example.hmm`. In both cases the `-p` parameter can be
used for invoking the parallelized version. In the first example the user
can optionally choose to save the data structure in eg. `example_out/`
using the `-o` parameter:

```
bin/forward -s example.seq -m example.hmm -e 500 -o example_out/
```

### generate\_hmm ###

Usage:
```
bin/generate_hmm <number of states> <alphabet size> <HMM filename>
```

Generates a random HMM with `<number of states>` states and `<alphabet size>` observables, and saves it to `<HMM filename>`.

### generate\_seq ###

Usage:
```
bin/generate_seq <HMM filename> <length> <observed sequence output filename> <state sequence output filename>
```

Given an HMM specified in `<HMM filename>`, runs the HMM for `<length>`
iterations and saves the resulting sequence of observables to
`<observed sequence output filename>` and the resulting sequence of
hidden states to `<state sequence output filename>`.