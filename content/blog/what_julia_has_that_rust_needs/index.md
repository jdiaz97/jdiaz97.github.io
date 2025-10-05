+++
authors = ["José Díaz"]
title = "What Julia has that Rust desperately needs"
description = "How community organizations solve a simple problem"
date = 2025-10-05
[taxonomies]
tags = ["Julia", "Rust", "Community"]
[extra]
toc = true
toc_inline = true
toc_ordered = true
+++

# The situation

The crate [ffmpeg-next](https://github.com/zmwangx/rust-ffmpeg) is a fork of the original and abandoned [ffmpeg](https://github.com/meh/rust-ffmpeg) crate. Now, we have new forks like [ffmpeg-the-third](https://github.com/shssoichiro/ffmpeg-the-third) and [rffmpeg](https://github.com/nrbnlulu/rffmpeg) that are more up-to-date. At the same time, ffmpeg-next is now trying to revive. So, which ffmpeg repo should I look at now? Sorry if reading that made you dizzy, but that's how bad the situation is.

[speculoos](https://github.com/oknozor/speculoos) is a fork of the unmaintained [spectral](https://github.com/cfrancia/spectral).

The great [ort](https://github.com/pykeio/ort) is a fork of [onnxruntime-rs](https://github.com/nbigaouette/onnxruntime-rs).

[juice](https://github.com/fff-rs/juice) is a fork of [leaf](https://github.com/autumnai/leaf).

[yaml-rust](https://github.com/chyh1990/yaml-rust) and [yaml-rust2](https://github.com/Ethiraric/yaml-rust2).

[serde-yaml](https://github.com/dtolnay/serde-yaml) and [serde-yaml-bw](https://github.com/bourumir-wyngs/serde-yaml-bw).

It is normal for people to drop open-source software; I'm not saying they should keep working forever. But it's sad that one person gets to gatekeep everyone else and say, "This is abandoned, I'll keep the crate name so now you have to make a worse one or be creative."

Anyway, so. What if all the disorganized effort was centralized and focused?

# How Julia solved this problem

The Julia community faced the exact same problem in the past. The solution is quite simple: let's move the packages into [self-organized GitHub organizations](https://julialang.org/community/organizations/).

- Packages for biology? You have [BioJulia](https://github.com/BioJulia)
- Packages for data? You have [JuliaData](https://github.com/JuliaData)
- Packages for TidyVerse-inspired packages? You have [TidierOrg](https://github.com/TidierOrg)

By doing this:

- The repos won't become inaccessible forever  
- You have other members (maybe) supporting your work
- We get less noise (fewer forks) and more productivity  
- We get a wonderful sense of community  

I think governance problems can emerge if people can't respect each other but the gains are worth considering. This is a HUGE positive for Julia, and I think Rust needs this too.

# FAQ

**Q:** This is not just a Rust problem btw, every programming language has this.  
**A:** They should fix it too. Just talking about what I see.