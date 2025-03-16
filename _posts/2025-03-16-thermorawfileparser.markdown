---
layout: post
title: Multiprocessing .RAW files with ThermoRawFileParser
date: '2025-03-16'
categories:
- Mass Spectrometry
- Code
tags:
- mass-spec
- proteomics
- multiprocessing
- code
---

Requirements
============
\>\> [Mono](https://www.mono-project.com/download/stable/#download-lin) for Linux


Motivation
============
We are currently in the process of moving our computational workflows over to Linux at the Slavov Lab. However, using [DIA-NN](https://github.com/vdemichev/DiaNN) in Linux with Thermo's native .RAW format is tricky, since the [.RAW file reader](https://thermo.flexnetoperations.com/control/thmo/login?nextURL=%2Fcontrol%2Fthmo%2Fdownload%3Felement%3D6306677) that is used by DIA-NN is only compatible with Windows. Now there is an updated version of the [.RAW file reader](https://github.com/thermofisherlsms/RawFileReader) which is cross-platform, and Vadim does intend to implement it in the [future](https://github.com/vdemichev/DiaNN/issues/1447). 

With the options available to us at the moment we can try either:
1. A workaround to run Windows applications like [Wine](https://www.winehq.org/).
2. Converting .RAW files to .mzML format using [ThermoRawFileParser](https://github.com/compomics/ThermoRawFileParser).

I went with the conversion option, since it is **less tedious**.

Approach
============
Follow the the recommended settings as described in the [DIA-NN GitHub](https://github.com/vdemichev/DiaNN?tab=readme-ov-file#raw-data-formats). The only thing to note is that compression is turned on by default and needs to be explicitly turned off, otherwise the search will have issues with the converted files.

However, I realized at this point that the way that ThermoRawFileParser iterates through each data file one at a time is simply not gonna scale with our needs. There doesn't seemed to be any arguement in the software to run parallel processes so I decided to implement my own.

Lucky for me, the problem seems to be [Embarrassingly Parallel](https://en.wikipedia.org/wiki/Embarrassingly_parallel). So we can simply write a script to parse our .RAW files from a data directory, and execute an independent process on each CPU core to convert the files in parallel. 

Multiprocessing Script
============
{% highlight bash %}
#!/bin/bash

DATA_DIR=${HOME}/Data/raw
OUTPUT_DIR=${HOME}/Data
export OUTPUT_DIR

# I use spack as an environment manager but you may not need this if mono is installed into your base environment
# Mono is required to run the parser
spack load mono


# -z must be passed! This disables compression.
# xargs -P parallelizes the conversion process so multiple files can be converted at once.
find "$DATA_DIR" -type f -name "*.raw" -print0 |
  xargs -0 -P "$(nproc)" -I % bash -c '
    mono "$HOME/ThermoRawFileParser/ThermoRawFileParser.exe" \
      -i="%" \
      -b="$OUTPUT_DIR/$(basename "%" .raw)" \
      -f=2 \
      -m=0 \
      -z
  '
{% endhighlight %}

**Performance Comparison:**

For testing I used 5 single-cell mixed-species runs from an Exploris 480, which is not a very in-depth test but it gets the main point across. Keep in mind that I'm testing this at home with an AMD Ryzen 5 3600 6-Core 3.60 GHz, so nothing fancy.

| Method             | Real Time | User Time | System Time |
| ------------------ | --------- | --------- | ----------- |
| Without Multiprocessing | 2m8.080s  | 3m1.319s  | 0m28.989s   |
| With Multiprocessing    | 0m42.016s | 4m1.692s  | 0m34.438s   |