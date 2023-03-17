# ELISA: Exit-Less, Isolated, and Shared Access

This document complements the paper "[Exit-Less, Isolated, and Shared Access for Virtual Machines](https://doi.org/10.1145/3582016.3582042)" presented at the International Conference on Architectural Support for Programming Languages and Operating Systems in March 2023 (ASPLOS'23).

## WARNING

- **The authors do not guarantee perfect security of the ELISA-relevant prototype implementations.**
- **The ELISA-relevant prototype implementations can break your entire system.**
- **The authors will not bear any responsibility if the implementations, provided by the authors, cause any problems.**

## Table of contents

- [Materials](#materials)
- [List of the ELISA prototype repositories](#list-of-the-elisa-prototype-repositories)
- [Requirements](#requirements)
- [Setup](#setup)
- [Trying an example application : elisa-app-nop](#trying-an-example-application--elisa-app-nop)
- [Commentary](#commentary)
	- [Section 4.1 : Anywhere Page Table (APT)](#section-41--anywhere-page-table-apt)
	- [Section 4.2 : Gate EPT Context](#section-42--gate-ept-context)
	- [Section 5.1 : ELISA-Specific Hypercalls](#section-51--elisa-specific-hypercalls)
	- [Section 5.2 : libelisa](#section-52--libelisa)
	- [Section 5.3 : Negotiation Steps](#section-53--negotiation-steps)
	- [Section 5.4 : Code for the Sub EPT Context](#section-54--code-for-the-sub-ept-context)
	- [Section 5.5 : Shared Memory Management](#section-55--shared-memory-management)
	- [Section 5.6 : Interrupt Setting](#section-56--interrupt-setting)
	- [Section 6.1 : Context Switch Overhead](#section-61--context-switch-overhead)
	- [Section 7.1 : VM Networking System](#section-71--vm-networking-system)

## Materials

- Paper: [https://doi.org/10.1145/3582016.3582042](https://doi.org/10.1145/3582016.3582042)
- Slides
- Lightning talk slides
- Lightning talk video: [https://www.youtube.com/watch?v=oLatL6TIIa4](https://www.youtube.com/watch?v=oLatL6TIIa4)
- Poster

