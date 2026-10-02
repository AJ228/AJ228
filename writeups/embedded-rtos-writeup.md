# Embedded Real-Time OS Kernel on ARM Cortex-M4

**Tools:** C, Assembly, GDB, ARM Cortex-M4 · **Context:** CMU Pittsburgh, Embedded Systems coursework, Fall 2025

## Overview

This was a two-person project to build a real-time operating system kernel for an ARM Cortex-M4
microcontroller from the ground up — starting below the level of an operating system entirely
(bootloader, reset handler) and building up through peripheral drivers, an application running on
a synchronous kernel, a full interrupt-driven rewrite, and finally preemptive multitasking with a
real scheduler. I worked on this with a partner; my primary ownership was the "dance party" application (both its
original synchronous version and the later interrupt-driven refactor), the scheduler and its
synchronization protocol, and two of the application modes in the final multitasking milestone,
alongside shared work on the driver and system-call layers.

The project handouts explicitly mention not to share their contents. As such, this writeup is
based on the project's scope and arc as I can describe it myself, not the assignment materials,
and stays general about implementation mechanics I can't independently verify against the
handouts — specific register-level detail, exact bug diagnoses, and so on are left out.

## Bootstrapping the kernel

The project started below the OS layer itself: a bootloader and reset handler that bring the
microcontroller up before any kernel code runs. Getting this right matters more than it might
seem from the outside — a mistake here doesn't throw a clean error, it just means nothing after
it works, and debugging has to happen with a debugger attached to real hardware rather than
through normal application-level tooling.

## Peripheral drivers

On top of that came drivers for GPIO, ADC, and I2C peripherals, built directly against
memory-mapped I/O. These were the building blocks everything later in the project depended on —
correctness here wasn't optional, since bugs at this layer would surface confusingly in whatever
application logic sat on top of it rather than at the point where they actually originated.

## Dance party: a synchronous application on the kernel

With drivers and a synchronous kernel in place, I built an application to validate the whole
stack: an adaptive LED system driven by a lux sensor and ADC input — nicknamed the "dance party"
milestone. This was less about the application logic itself and more about proving the underlying
kernel and drivers actually worked together correctly under something resembling real use, rather
than just passing isolated driver tests.

## From synchronous to interrupt-driven

The kernel, and then the dance party application built on top of it, were each separately
refactored from synchronous to interrupt-driven operation — the kernel refactor as shared work,
and the dance party refactor as my own. Doing this in two passes — once for the kernel, then again
for the application that depended on it — made clear how much of a synchronous design's
assumptions are implicit rather than written down anywhere: control flow that "just works" when
everything happens in order breaks in new ways once execution can be interrupted mid-operation,
and a lot of the refactor was finding those implicit assumptions by hitting the bugs they caused.

## Multitasking

The final phase built real preemptive multitasking on top of the interrupt-driven kernel: a
system-call interface to separate user-space and kernel-space execution, then context switching
and scheduling to actually run multiple tasks concurrently. My primary contribution here was the
scheduler — a preemptive, priority-based design using rate-monotonic scheduling, paired with the
priority ceiling protocol to bound priority inversion when tasks share resources. Getting a
real-time scheduler correct on real hardware is a different problem than reasoning about it on
paper; the textbook guarantees assume clean preemption boundaries, and a meaningful part of the
work was making the implementation hold up under real interrupt timing rather than an idealized
task model.

The project culminated in a multitasking system supporting four application modes running on this
scheduler. I built two of them: a traffic light simulator, and a custom LED controller where user
input drove the color and brightness of the main LED in real time.

## Output

The end deliverable was a working preemptive RTOS on real ARM Cortex-M4 hardware, built up in
stages from bootloader to full multitasking: GPIO/ADC/I2C drivers, an interrupt-driven kernel, a
rate-monotonic scheduler with priority-ceiling synchronization, a user/kernel system-call
boundary, and a four-mode multitasking application layer running concurrently on top of it all.
