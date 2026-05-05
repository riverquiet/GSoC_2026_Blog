# Introduction

This is the blog for GSoC 2026.  
Contents would be the process of RTEMS Controller Area Network (CAN) Stack Improvement: BeagleBone Black D-CAN Driver Implementation.

## Project Abstract
This project develops a D-CAN driver for the AM335x controller on the BeagleBone Black and integrates it into the RTEMS CAN stack, enabling reliable CAN communication through standard interfaces such as /dev/canX. The driver will support FIFO-based reception, priority-aware transmission, and real-time behavior.  

By adding CAN support to a widely used embedded platform, this project extends RTEMS usability in applications such as robotics, automotive systems, and industrial control. The implementation will be validated on real hardware and follow RTEMS coding standards, while also serving as a reference for future CAN driver development.

## Project Description
This project focuses on implementing support for the AM335x D-CAN controller on the BeagleBone Black within RTEMS. While RTEMS provides a generic CAN/CAN FD stack and user-facing device interface, it currently lacks a driver for this hardware. This project will enable RTEMS applications to access CAN functionality through standard device nodes such as /dev/canX.  

The D-CAN controller uses a mailbox-based hardware architecture, where each message object is configured individually for transmission or reception. In contrast, the RTEMS CAN stack follows a queue-based abstraction. Bridging this mismatch is a key design challenge.  

To address this, the driver will introduce a mapping layer between hardware and software:  
    - RX Path: A subset of hardware message objects will be configured to emulate FIFO behavior. The driver will extract messages and push them into RTEMS CAN queues while preserving ordering.  
    - TX Path: A software-managed queue will group messages by priority. Hardware mailboxes will be dynamically assigned based on availability, allowing higher-priority messages to preempt lower-priority ones while maintaining FIFO order within the same priority class.  
    - Synchronization: Register access via IFx interfaces and interrupt handling will be carefully managed to ensure consistency and avoid race conditions.  

The implementation will follow an incremental approach. The initial stage will establish a minimal working driver with initialization, polling-based transmission, and FIFO-based reception. After validating correctness, the driver will be extended to support interrupt-driven operation, improved mailbox utilization, and priority-aware scheduling.  

Existing implementations such as the Linux c_can driver will be studied for design reference, but not directly reused. Validation will be performed on real hardware using loopback and dual-board communication tests (RTEMS ↔ Linux).  

The final outcome will be a robust, well-structured, and maintainable D-CAN driver integrated into the RTEMS CAN stack, providing a strong foundation for future enhancements such as performance optimization and optional DMA support.


## Blog setup
This blog uses mdBook + github.io.  
I installed the mdBook using homebrew, which I used before, so it is just easy to continue to use that. Other resources as below:  
* [Document tutorial](https://mdbook.code-maven.com/github-pages-with-mdbook)
* [Youtube tutorial](https://www.youtube.com/watch?v=x3vF9YiWBMQ)
* [Official mdBook](https://rust-lang.github.io/mdBook/)
* [Connect to github with ssh](https://docs.github.com/en/authentication/connecting-to-github-with-ssh)
* [Markdown Example](https://gist.github.com/allysonsilva/85fff14a22bbdf55485be947566cc09e)

## About me
This is Ning. I just got my master degree in Electrical Engineering in the Internet of Things from University at Buffalo, SUNY.   
And I will pursue my PhD degree in the cross area of embedded system and cybersecurity in 2026 Fall in University of Colorado Colorado Springs.  

The Internet of Things is naturally close to embedded system applications and I have built some embedded applications before and it was interesting. I like the process of solving problems. The difficulties will be the great opportunities for learning and development. I am looking forward this summer to be challenged and learn and grow.  
I found the documentation in RTEMS website is very helpful, which helps me understand how to start at the very beginning. And through the responses and resources with the community helps know the next steps. Really thankful to the mentors, who reply promptly and give their valuable insights.  





