# NaxRiscv Documentation

This Github repository presents a non-official documentation of [NaxRiscv](https://github.com/SpinalHDL/NaxRiscv) core plugins.

For the official documentation, please follow this link :

https://spinalhdl.github.io/NaxRiscv-Rtd/main/NaxRiscv/introduction/index.html

## Installation

This documentation is built around several markdown files. To read them, you need [Obsidian](https://obsidian.md/).

This software allows you to create links between files and use plugins such as Excalidraw for diagrams.

After installing Obsidian, and cloning this repository, you need to open the folder as a new vault, and normally the necessary plugins will already be installed.

## Documentation structure

There are two folders :

- **NaxRiscv Plugins** : where the core plugins are documented, for each plugin there is 3 files.
    - **0. Presentation** : introduces the plugin and gives a few explanations about it
    - **1. Environment** : shows links with others plugins
    - **2. Internal** : gives a schematic of the plugin's interior

In the diagrams, links take you directly to the correct chapter in the associated `0. Presentation`.
***Without clicking, but just by hovering over the link while pressing Ctrl, a tooltip will appear in the diagram.***

- **OoO Basics** : based on a [Georgia Tech course](https://www.udacity.com/course/high-performance-computer-architecture--ud007), this is a reminder of how an out-of-order core works.
> It is strongly recommended to see the OoO Basics folder and the corresponding chapters in the online course before reading the NaxRiscv code and documentation.


The two others files are :
- **NaxRiscv Pipelines** : gives an overview of the NaxRiscv pipeline from the fetch stages to the execution stages. (Under construction)
- **OoO core schematic vs NaxRiscv** : gives a parallel between the NaxRiscv core and the theoretical diagram of an OoO core(`OoO Basics/0. OoO core schematic`). 