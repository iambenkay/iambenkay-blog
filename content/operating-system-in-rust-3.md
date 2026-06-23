+++
title = "Building an Operating System In Rust Part 3"
slug = "building-an-operating-system-in-rust-part-3"
date = 2026-06-20
description = "When running programs on an existing operating system, there are a lot of things that are handled under the hood for us by the operating system. They are invisible to us and we take them for granted, we expect them to just work. In kernel space there are no training wheels"

[extra]
header_img = "/images/header-rust.jpg"
header_alt = "Rust"
section = "Programming"
word_count = 3193
author = "Benjamin Chibuzor-Orie"
tags = ["Rust", "Operating System", "Low Level"]

[taxonomies]
series = ["Building an Operating System in Rust"]
+++
In [Part 2 we built an entrypoint for our operating system](@/operating-system-in-rust-2.md) and ran it successfully on QEMU and a physical Raspberry Pi 5 device. We also learnt about the boot image and how to link our kernel source code into one kernel binary. Next we are going to write to the screen!

If you have been binging this series, now is the time to take a short break before going into the next part because it's the most complex part yet. In high level land, writing to the screen is a trivial task: you want text then just call `println`, you want shapes then just use a drawing library like `bevy` or `wgpu`. T