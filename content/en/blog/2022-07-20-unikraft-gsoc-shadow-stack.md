+++
title = "GSoC'22: Shadow Stack"
date = "2022-07-20T13:00:00+01:00"
author = "Maria Sfiraiala"
tags = ["GSoC'22"]
+++

<img width="100px" src="https://summerofcode.withgoogle.com/assets/media/gsoc-2022-badge.svg" align="right" />

# GSoC'22: Shadow Stack

While the [previous blog post](https://unikraft.org/blog/2022-07-05-unikraft-gsoc-shadow-stack/) described the first steps took into the direction of familiarizing myself with Unikraft and an initial attempt to using `clang's` ShadowCallStack, in this post, we will take a look into some implementations that were tried in the meantime.

## Starting point

Following the findings of previous weeks of investigation work, the issue of `clang` not providing a runtime for the ShadowCallStak was resolved by coming up with a primitive [constructor](https://gist.github.com/mariasfiraiala/60389dd16fef0fdc11d7f7972e320a9a) which would take care of initializing the Shadow Stack.

This constructor had to be moved in the `ukboot` library, more specifically, in `boot.c`, in order to be called at the same time (at bootstrapping) with the other Unikraft specific constructors and init functions.

As a result, the constructor cannot maintain its current form, as it brings critical overhead to the boot process, which, following Unikraft's vision, should be highly performant.

Documenting each and every step took by this stage was a must, and you'll be able to find my investigation work [here](https://github.com/mariasfiraiala/scs-work/tree/master/unikraft-scs); understanding the layout of both `boot.c` and `traps_arm64.c` facilitated debugging work.

## Progress

The biggest milestone achieved until now was bringing Shadow Stack support for a simple app (`helloworld`, to be more precise). 

This constitutes the key part of the [draft PR](https://github.com/unikraft/unikraft/pull/505) and the whole documentation of the process can be found [here](https://github.com/mariasfiraiala/scs-work/blob/master/unikraft-scs/unikraft-scs-for-helloworld.md).

As a Proof of Concept, I used both `gdb` to investigate the content of the Shadow Stack and `assembly` code to prove the way the prologue and epilogue of the function is modified by this security mechanism.

What's more, improving the performance of the aforementioned constructor was anticipated and a first step taken into this direction was giving the Shadow Stack the same size as the regular, visible stack.

## Problems

The biggest problem faced during this stage was debugging my Unikraft application; catching traps isn't fun, but the community helped and a [straightforward solution](https://gist.github.com/mariasfiraiala/34a7b5b41c4e5515c7f0ad8a2c220ef9) to debugging an `AArch64` app was documented.

Nevertheless, not realizing that the `x18` register wasn't fixed (using the `-ffixed-x18` flag) for all Unikraft functions was another issue which was finally resolved with help from my mentors.

## Interesting findings

At the request of my mentors, I embarked on a mission to document the reason as to why the `x86` `clang's` version of the ShadowCallStack was dropped.

The reason seems to be quite obscure, but after putting the pieces together, I was left with the conclusion that a Time-Of-Check-To-Time-Of-Use (TOCTOU) vulnerability was the main actor in this complicated play.

But more on that in [my documentation](https://github.com/mariasfiraiala/scs-work/blob/master/utils/doc-scs-arm-vs-x86.md).

## Next steps

Testing the Shadow Stack support (which proved to work fine on the `helloworld` app) on other, more complex applications (such as `SQLite`, `redis` and `nginx`) is an important first step forward.

I've already started working on these apps, here it is a [sneak peak](https://github.com/mariasfiraiala/scs-work/blob/master/unikraft-scs/unikraft-scs-for-complex-apps.md):

| App\Compiler | gcc - x86 | gcc - aarch64 | clang - x86 | clang - aarch64 | clang with scs | gcc-12 with scs |
|--------------|-----------|---------------|-------------|-----------------|----------------|-----------------|
| SQLite | :heavy_check_mark: | :soon: | :soon: | :soon: | :soon: | :soon: |
| redis | :heavy_check_mark: | :soon: | :soon: | :soon: | :soon: | :soon: |
| nginx | :heavy_check_mark: | :soon: | :soon: | :soon: | :soon: | :soon: |

The second step consists of providing a simple ROP attack and observing how the execution behaves under these circumstances.

Thirdly, it should also be taken into consideration how my implementation would influence future security mechanisms (CET or CFI).

I aim to provide a proposal which will facilitate fitting both Shadow Stack and other related means of securing against ROP attacks together, on Unikraft.