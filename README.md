# FPGA Programming via Raspberry Pi 5 (JTAG over GPIO) | Programmation d'un FPGA via Raspberry Pi 5

*Programming a Xilinx Artix-7 100T from a Raspberry Pi 5, using only GPIO pins as a JTAG interface, without Vivado or a dedicated JTAG cable.*

*Programmation d'un Xilinx Artix-7 100T depuis une Raspberry Pi 5, en utilisant uniquement les broches GPIO comme interface JTAG, sans Vivado ni câble JTAG dédié.*

**University project / Projet universitaire**: Université de Bourgogne Europe, L3 SPI Électronique, 2025-2026

---

## 🇬🇧 English

### Description

The goal is to let a Raspberry Pi 5 program and update an FPGA on its own (for a future FPGA daughter board), with no external computer and no official Xilinx tools.

Two approaches were implemented and compared:

1. **Python bit-banging**: the JTAG protocol (TCK, TMS, TDI, TDO) is generated in software with `lgpio`. This shows exactly what happens on the wire.
2. **openFPGALoader**: an existing open-source C++ tool, compiled from source on the Pi with the `libgpiod` backend (required by the RP1 I/O chip of the Pi 5).

This project demonstrates:

- JTAG protocol and the 16-state TAP machine (IEEE 1149.1)
- Xilinx 7-series configuration sequence (`JPROGRAM`, `CFG_IN`, `JSTART`)
- Bitstream file structure (`.bit` header, sync word `0xAA995566`) and Intel HEX `.mcs`
- Linux tooling: GPIO access (`lgpio`, `libgpiod`), building from source with CMake
- Volatile (SRAM) vs non-volatile (QSPI flash) FPGA configuration

### Hardware

| Component | Model |
| --- | --- |
| SBC | Raspberry Pi 5 (RP1 I/O controller, 3.3 V GPIO) |
| FPGA board | Digilent Nexys A7 (Xilinx Artix-7 XC7A100T) |
| Flash | Spansion S25FL128S (128 Mb QSPI), on the Nexys A7 |

No level shifter is needed: the Pi's 3.3 V GPIO is compatible with the Artix-7 JTAG levels.

### Wiring

| Raspberry Pi 5 | JTAG signal | Nexys A7 |
| --- | --- | --- |
| GPIO 16 (Pin 36) | TDI | TDI |
| GPIO 9 (Pin 21) | TDO | TDO |
| GPIO 17 (Pin 11) | TCK | TCK |
| GPIO 24 (Pin 18) | TMS | TMS |
| GND (Pin 6) | GND | GND |

### JTAG programming sequence

1. TAP reset (TMS = 1 for several clock cycles)
2. Read IDCODE, check it is `0x13631093` (XC7A100T)
3. `JPROGRAM` (0x0B): erase current configuration, then wait
4. `CFG_IN` (0x05): shift the bitstream through TDI
5. `JSTART` (0x0C): start the FPGA, extra clocks
6. Check the `DONE` LED

### Method 1: Python bit-banging

```bash
sudo apt install python3-lgpio
python3 python/fpga_program.py design.bit
```

The script finds the sync word `0xAA995566` in the `.bit` file, skips the header, and shifts the configuration data bit by bit.

| Parameter | Value |
| --- | --- |
| Bitstream size | 3,825,896 bytes (config data: 3,825,740 bytes) |
| Bits transferred | 30,605,920 |
| Speed | ~0.30 Mbps |
| Programming time | 102.7 s |

The limit comes from the number of `lgpio` calls per bit (about 6) and Python overhead.

> **Note on AI assistance:** the Python script was developed with the help of an AI assistant (Claude, Anthropic), then read, tested and validated on the real board. The full report is in `docs/`.

### Method 2: openFPGALoader

Build from source (it is not in the Raspberry Pi OS apt repositories):

```bash
sudo apt update
sudo apt install -y git cmake pkg-config libusb-1.0-0-dev libudev-dev \
    libftdi1-dev libhidapi-dev zlib1g-dev libgpiod-dev libgpiod2
git clone https://github.com/trabucayre/openFPGALoader
cd openFPGALoader && mkdir build && cd build
cmake .. -DENABLE_GPIOD=ON
make -j4 && sudo make install
```

Usage (`--pins` order is **TDI:TDO:TCK:TMS**):

```bash
# Detect the FPGA
sudo openFPGALoader -c libgpiod --fpga-part xc7a100t --pins 16:9:17:24 --detect

# Volatile programming (.bit -> SRAM)
sudo openFPGALoader -c libgpiod --fpga-part xc7a100t --pins 16:9:17:24 design.bit

# Non-volatile programming (.mcs -> QSPI flash)
sudo openFPGALoader -c libgpiod --fpga-part xc7a100t --pins 16:9:17:24 -f design.mcs
```

For `.mcs`, the FPGA has no direct JTAG access to its flash. openFPGALoader first loads a bridge design (`spiOverJtag`) into the FPGA SRAM over JTAG. That bridge gives access to the QSPI flash, which is then bulk-erased and written.

### Results

- IDCODE `0x13631093` read correctly by both methods
- `.bit` programming works with both methods (test design: a VHDL LED controlled by a switch, built in Vivado)
- `.mcs` programming works with openFPGALoader; the FPGA reconfigures itself from flash at power-up
- Flash chip detected automatically (S25FL128S)

### Challenges & solutions

| Problem | Solution |
| --- | --- |
| `cmake` run from the wrong directory | Run it from `openFPGALoader/build/` |
| GPIOD support disabled by default | Force `-DENABLE_GPIOD=ON` |
| Cable `gpiod` not found | Use `-c libgpiod` |
| Wrong pin order | `--pins` expects TDI:TDO:TCK:TMS |
| Old GPIO libraries do not work on Pi 5 | Use `lgpio` / `libgpiod` (RP1 chip) |

### Limitations & future work

- Python bit-banging is slow (0.30 Mbps); rewriting the shift loop in C is the obvious next step
- No readback / verification of the bitstream after programming yet
- Integrate openFPGALoader in an automatic update script for the daughter board
- Measure and document openFPGALoader's actual programming time (not measured in the report)

### Repository structure

```
├── python/
│   └── fpga_program.py      # JTAG bit-banging script (lgpio)
├── vhdl/                    # LED/switch test design + constraints
├── docs/
│   └── rapport_projet.pdf   # Full project report (French)
├── images/                  # Wiring photo, terminal screenshots
└── README.md
```

---

## 🇫🇷 Français

### Description

L'objectif est qu'une Raspberry Pi 5 puisse programmer et mettre à jour un FPGA de façon autonome (pour une future carte-fille FPGA), sans ordinateur externe ni outils officiels Xilinx.

Deux approches ont été réalisées et comparées :

1. **Bit-banging Python** : le protocole JTAG (TCK, TMS, TDI, TDO) est généré par logiciel avec `lgpio`. Cela permet de voir précisément ce qui passe sur le bus.
2. **openFPGALoader** : outil open source en C++ existant, compilé depuis les sources sur la Pi avec le backend `libgpiod` (nécessaire pour la puce d'E/S RP1 de la Pi 5).

Ce projet illustre :

- Le protocole JTAG et la machine d'états TAP à 16 états (IEEE 1149.1)
- La séquence de configuration Xilinx 7-series (`JPROGRAM`, `CFG_IN`, `JSTART`)
- La structure d'un fichier `.bit` (en-tête, mot de synchronisation `0xAA995566`) et du format Intel HEX `.mcs`
- L'environnement Linux : accès GPIO (`lgpio`, `libgpiod`), compilation depuis les sources avec CMake
- Configuration volatile (SRAM) et non volatile (flash QSPI)

### Matériel et câblage

Voir les tableaux de la section anglaise. Niveaux logiques 3,3 V compatibles : aucun adaptateur nécessaire.

### Méthode 1 : bit-banging Python

Le script localise le mot de synchronisation `0xAA995566`, ignore l'en-tête et envoie les données de configuration bit par bit (30 605 920 bits, ~0,30 Mbps, 102,7 s).

> **Note :** le script Python a été développé avec l'aide d'un assistant IA (Claude, Anthropic), puis relu, testé et validé sur la carte réelle.

### Méthode 2 : openFPGALoader

Compilation depuis les sources avec `-DENABLE_GPIOD=ON`, puis programmation `.bit` (SRAM, volatile) ou `.mcs` (flash QSPI, permanente) avec les commandes ci-dessus. Pour le `.mcs`, l'outil charge d'abord un pont `spiOverJtag` dans la SRAM du FPGA, ce qui donne accès à la flash QSPI (effacement complet puis écriture).

### Résultats

- IDCODE `0x13631093` détecté correctement par les deux méthodes
- Programmation `.bit` réussie (design de test VHDL : LED commandée par un interrupteur)
- Programmation `.mcs` réussie avec openFPGALoader, reconfiguration automatique au démarrage
- Flash S25FL128S détectée automatiquement

### Perspectives

- Réécrire la boucle de transfert en C pour accélérer le script
- Vérifier le bitstream après programmation
- Intégrer openFPGALoader dans un script de mise à jour automatique de la carte-fille

---

## Authors / Auteurs

- **Wassim Kessaissia**: [@wassimkessaissia](https://github.com/wassimkessaissia)
- **Oussama Haithem Messabih**

Supervisor / Encadrant: M. Denis Pellion

## References / Références

- [openFPGALoader](https://github.com/trabucayre/openFPGALoader): G. Goavec-Merou
- Xilinx UG470: *7 Series FPGAs Configuration User Guide*
- Digilent *Nexys A7 Reference Manual*
- [lgpio](https://github.com/joan2937/lgpio)

## License

Add a `LICENSE` file (MIT, like your other repos) before publishing.

