# basicpitch.cpp

C++20 inference for the [Spotify basic-pitch](https://github.com/spotify/basic-pitch) automatic music transcription/MIDI generator neural network with ONNXRuntime, Eigen, and libremidi. Demo apps are provided for WebAssembly/Emscripten and a cli app.

I use [ONNXRuntime](https://github.com/microsoft/onnxruntime) and scripts from the excellent [ort-builder](https://github.com/olilarkin/ort-builder) project to implement the neural network inference like so:
* Convert the ONNX model to ORT (onnxruntime)
* Include only the operations and types needed for the specific neural network, cutting down code size
* Compile the model weights to a .c and .h file to include it in the built binaries

After the neural network inference, I use [libremidi](https://github.com/celtera/libremidi) to replicate the end-to-end MIDI file creation of the real basic-pitch project. I didn't run any official measurements but the WASM demo site is **much faster** than Spotify's own [web demo](https://basicpitch.spotify.com/).

## Project design

* [ort-model](./ort-model) contains the model in ONNX form, ORT form, and the generated h and c file
* [scripts](./scripts) contain the ORT model build scripts
* [src](./src) is the shared inference and MIDI creation code
* [src_wasm](./src_wasm) is the main WASM function, used in the web demo
* [src_cli](./src_cli) is a Linux cli app (for debugging purposes) that uses [libnyquist](https://github.com/ddiakopoulos/libnyquist) to load the audio files
* [vendor](./vendor) contains third-party/vendored libraries
* [web](./web) contains basic HTML/Javascript code to host the WASM demo

## Usage

I recommend the tool `midicsv` for inspecting MIDI events in CSV format without more complicated MIDI software, to compare the files output by basicpitch.cpp to the real basic-pitch.


### CLI app

After following the build instructions below:
```
$ ./build/build-cli/basicpitch ~/Downloads/clip.wav ./midi-out-cpp
basicpitch.cpp Main driver program
Predicting MIDI for: /home/sevagh/Downloads/clip.wav
Input samples: 441000
Length in seconds: 10
Number of channels: 2
Resampling from 44100 Hz to 22050 Hz
output_to_notes_polyphonic
note_events_to_midi
Before iterating over note events
After iterating over note events
Now creating instrument track
done!
MIDI data size: 889
Wrote MIDI file to: "./midi-out-cpp/clip.mid"
```


## Build instructions

(Only tested on Windows 10). I'm assuming you have a typical C/C++ toolchain e.g. make, cmake, gcc/g++, for your OS.
Clone the repo with submodules: 
```
$ git clone  https://github.com/andrewperezledo/basicpitch.cpp
```


**Optional:** if you want to re-convert the ONNX model to ORT in the ort-model directory, use `scripts/convert-model-to-ort.sh ./ort-models/model.onnx`. The ONNX model is copied from `./vendor/basic-pitch/basic_pitch/saved_models/icassp_2022/nmp.onnx`

(New) Build cli app:
```
$ cd src_cli
$ mkdir build-cli
$ cd build-cli
$ cmake ../src_cli -G "Visual Studio 17 2022" -A x64
$ cmake --build . --config Release
```

(OLD) Build cli app:
```
$ make cli
$ ls build/build-cli/basicpitch
build/build-cli/basicpitch
```

For WebAssembly, first, set up the [Emscripten SDK](https://github.com/emscripten-core/emsdk). Then, build the WASM app with your EMSDK env script:
```
$ export EMSDK_ENV_PATH=/path/to/emsdk/emsdk_env.sh
$ make wasm
$ ls build/build-wasm/basicpitch.wasm
build/build-wasm/basicpitch.wasm
```

This also copies the updated `basicpitch.{wasm,js}` to the `./web` directory.
