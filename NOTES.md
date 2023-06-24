*** SPDIF digital mixer ***
HW requirements:
 - Hifiberry Digi IO
 - Raspberry Pi 3B
SW requirements:
 - Raspberry Pi OS
 - pipewire
 - pareceive


*** Raspberry Pi OS ***
Raspberry pi os based on debian; as of May 3rd 2023 Debian version: 11 (bullseye)
That can be checked in /etc/os-release

*** Pipewire ***
It needs to be installed from the backports source tree:
```
  echo "deb http://deb.debian.org/debian bullseye-backports main contrib non-free" | sudo tee /etc/apt/sources.list.d/bullseye-backports.list
  sudo apt update
  sudo apt -t bullseye-backports install pipewire wireplumber
```

It can be tested with mplayer: `mplayer some-stereo-audio-file.mp3 -vo null -ao pulse`

A virtual 5.1 to stereo mapping interface needs to be created in `~/.config/pipewire/pipewire.conf.d/surround-51-to-stereo.conf`
```
# Create a 5.1 surround to 2.0 stereo device.
# The general idea is that you need to send inputs to outputs.
# Each channel's volume can be controlled independently by tweaking the Gain of each channel of both mixers:
#  pw-cli ls Node # find this device's ID
#  pw-cli s ${NODE_ID} Props '{ params = [ "mixL:Gain 2" 3.0 "mixR:Gain 2" 3.0 ] }' # put gain at 3.0 on the center channel for L and R.
#  pw-cli e ${NODE_ID} Props # check the changes

context.modules = [
    {   name = libpipewire-module-filter-chain    # Create a "filter-chain" device.
        args = {
            # Those names are visible to the outside world
            node.description = "Surround 5.1 to Stereo"
            media.name       = "Surround 5.1 to Stereo"

            filter.graph = {
                nodes = [
                    # Create "copy" filters, one for each input.
                    { name = IFL  type = builtin label = copy }
                    { name = IFR  type = builtin label = copy }
                    { name = IFC  type = builtin label = copy }
                    { name = ILFE type = builtin label = copy }
                    { name = ISL  type = builtin label = copy }
                    { name = ISR  type = builtin label = copy }

                    # Create "mixer" filters, one for each output.
                    { name = mixL type = builtin label = mixer }
                    { name = mixR type = builtin label = mixer }
                ]
                links = [
                    # Copy the 2 input left channels + the front and bass channels to the left mixer.
                    { output = "IFL:Out"  input = "mixL:In 1" } # send the output of IFL (as it's a "copy" filter, the output is the same as the input) to mixL's 1st channel input
                    { output = "IFC:Out"  input = "mixL:In 2" }
                    { output = "ILFE:Out" input = "mixL:In 3" }
                    { output = "ISL:Out"  input = "mixL:In 4" }

                    # Copy the 2 input right channels + the front and bass channels to the right mixer.
                    { output = "IFR:Out"  input = "mixR:In 1" }
                    { output = "IFC:Out"  input = "mixR:In 2" }
                    { output = "ILFE:Out" input = "mixR:In 3" }
                    { output = "ISR:Out"  input = "mixR:In 4" }
                ]

                # Declare the "copy" filters as the inputs. Must always be "...:In" here.
                inputs  = [ "IFL:In" "IFR:In" "IFC:In" "ILFE:In" "ISL:In" "ISR:In" ]

                # Declare the "mix*" filters as the outputs. Must always be "...:Out" here.
                outputs = [ "mixL:Out" "mixR:Out"]
            }

            # Declare that this device has 6 inputs.
            capture.props = {
                node.name         = "input_surround_51"
                audio.position    = [ FL FR FC LFE SL SR ] # Those names are visible to the outside world
                media.class       = "Audio/Sink"
            }

            # Declare that this device has 2 outputs.
            playback.props = {
                node.name         = "output_stereo"
                audio.position    = [ FL FR ] # Those names are visible to the outside world
                stream.dont-remix = true
                node.passive      = true
            }
        }
    }
]
```

then restart pipewire: `systemctl --user restart pipewire.service`

The new interface can be tested with

`mplayer some-51-audio-file.ac3 -vo null -ao pulse::input_surround_51`


*** pareceive ***

Clone it from `https://github.com/lorban/pareceive` then build it. These libs will be needed:

`sudo apt install libavformat-dev pulseaudio-dev`

It needs a patched `libavformat`. That lib is part of this debian package: `https://packages.debian.org/bullseye/libavformat58` which is built from the ffmpeg sources: `http://security.debian.org/debian-security/pool/updates/main/f/ffmpeg/ffmpeg_4.3.6.orig.tar.xz`
Download and build ffmpeg then apply the patches available at https://patchwork.ffmpeg.org/project/ffmpeg/list/?submitter=1191 to libavformat. Then copy the new binary to `/lib/aarch64-linux-gnu/libavformat.so.58.custom` and redo the `libavformat.so` and `libavformat.so.58` symlinks.


