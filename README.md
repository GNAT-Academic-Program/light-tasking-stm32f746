# Light-Tasking Runtime for STM32F746

A light-tasking Ada runtime for the STM32F746 SoC, providing bare-metal support with configurable clock tree and interrupt handling.

## Overview

This runtime provides a minimal tasking profile for STM32F746 microcontrollers, suitable for embedded applications that require basic concurrency support without the full overhead of the Ravenscar profile.

**Version:** 15.0.0-dev  
**License:** GPL-3.0-or-later WITH GCC-exception-3.1  
**Target:** ARM Cortex-M7 (STM32F746)

## Features

- Light-tasking profile for embedded Ada applications
- Configurable clock tree with PLL support
- Support for HSI (High-Speed Internal) and HSE (High-Speed External) oscillators
- Configurable interrupt stack sizes
- Default 216 MHz system clock configuration
- Compile-time validation of PLL configurations

## Installation

### Using Alire

Add the runtime as a dependency in your project's `alire.toml`:

```toml
[[depends-on]]
light_tasking_stm32f746 = "*"
```

Then pin the runtime to the GitHub repository:

```toml
[[pins]]
light_tasking_stm32f746 = { url = "https://github.com/GNAT-Academic-Program/light_tasking_stm32f746.git", branch = "main" }
```

### Project File Configuration

Edit your GPR project file to include the runtime:

```ada
with "runtime_build.gpr";

project Your_Project is
   for Target use runtime_build'Target;
   for Runtime ("Ada") use runtime_build'Runtime ("Ada");
   
   -- Your project configuration here
   
   package Linker is
      for Switches ("Ada") use Runtime_Build.Linker_Switches;
   end Linker;
end Your_Project;
```

## Clock Configuration

### Default Configuration

By default, the runtime configures:
- **System Clock (SYSCLK):** 216 MHz from PLL
- **PLL Source:** HSI (16 MHz internal oscillator)
- **PLL Configuration:**
  - VCO Input: HSI / 16 = 1 MHz
  - VCO Output: 1 MHz × 432 = 432 MHz
  - SYSCLK (PLLP): 432 MHz / 2 = 216 MHz
  - USB/SDIO/RNG (PLLQ): 432 MHz / 9 = 48 MHz
- **AHB Prescaler:** DIV1 (216 MHz)
- **APB1 Prescaler:** DIV4 (54 MHz)
- **APB2 Prescaler:** DIV2 (108 MHz)

### Custom Clock Configuration

Configure the clock tree using Alire configuration variables in your `alire.toml`:

#### Example: 216 MHz from 8 MHz HSE Crystal

```toml
[configuration.values]
# Configure an 8 MHz HSE crystal oscillator
light_tasking_stm32f746.HSE_Clock_Frequency = 8000000
light_tasking_stm32f746.HSE_Bypass = false

# Select PLLCLK as the SYSCLK source
light_tasking_stm32f746.SYSCLK_Src = "PLLCLK"

# Configure the PLL VCO to run at 432 MHz from the 8 MHz HSE
# fVCO = fHSE × (N/M) = 8 MHz × (432/8) = 432 MHz
light_tasking_stm32f746.PLL_Src = "HSE"
light_tasking_stm32f746.PLL_N_Mul = 432
light_tasking_stm32f746.PLL_M_Div = 8

# Configure SYSCLK (PLLP) to run at 216 MHz from the 432 MHz VCO
light_tasking_stm32f746.PLL_P_Div = "DIV2"

# Configure PLLQ to run at 48 MHz for USB/SDIO/RNG domains
light_tasking_stm32f746.PLL_Q_Div = 9

# Configure AHB/APB clocks
light_tasking_stm32f746.AHB_Pre  = "DIV1"
light_tasking_stm32f746.APB1_Pre = "DIV4"
light_tasking_stm32f746.APB2_Pre = "DIV2"
```

### Available Configuration Variables

#### Oscillator Configuration

| Variable | Type | Range/Values | Default | Description |
|----------|------|--------------|---------|-------------|
| `LSI_Enabled` | Boolean | true/false | true | Enable Low-Speed Internal oscillator |
| `LSE_Enabled` | Boolean | true/false | false | Enable Low-Speed External oscillator |
| `LSE_Bypass` | Boolean | true/false | false | Bypass LSE oscillator (external clock) |
| `HSE_Bypass` | Boolean | true/false | false | Bypass HSE oscillator (external clock) |
| `HSE_Clock_Frequency` | Integer | 4000000-26000000 | 8000000 | HSE frequency in Hz |

#### PLL Configuration

| Variable | Type | Range/Values | Default | Description |
|----------|------|--------------|---------|-------------|
| `PLL_Src` | Enum | HSE, HSI | HSI | PLL input source |
| `PLL_M_Div` | Integer | 2-63 | 16 | PLL input divider (VCO input = source / M) |
| `PLL_N_Mul` | Integer | 50-432 | 432 | PLL multiplier (VCO output = input × N) |
| `PLL_P_Div` | Enum | DIV2, DIV4, DIV6, DIV8 | DIV2 | Main PLL divider for SYSCLK |
| `PLL_Q_Div` | Integer | 2-15 | 9 | PLL divider for USB/SDIO/RNG (48 MHz) |
| `PLL_Q_Enable` | Boolean | true/false | true | Enable PLL Q output |

**PLL Constraints:**
- VCO input frequency (source / M): 1-2 MHz
- VCO output frequency (input × N): 100-432 MHz
- The runtime validates these constraints at compile time

#### System Clock Configuration

| Variable | Type | Range/Values | Default | Description |
|----------|------|--------------|---------|-------------|
| `SYSCLK_Src` | Enum | HSI, HSE, PLLCLK | PLLCLK | System clock source |
| `AHB_Pre` | Enum | DIV1, DIV2, DIV4, DIV8, DIV16, DIV64, DIV128, DIV256, DIV512 | DIV1 | AHB prescaler |
| `APB1_Pre` | Enum | DIV1, DIV2, DIV4, DIV8, DIV16 | DIV4 | APB1 prescaler (max 54 MHz) |
| `APB2_Pre` | Enum | DIV1, DIV2, DIV4, DIV8, DIV16 | DIV2 | APB2 prescaler (max 108 MHz) |

#### Interrupt Stack Configuration

| Variable | Type | Range | Default | Description |
|----------|------|-------|---------|-------------|
| `Interrupt_Stack_Size` | Integer | ≥1 | 1024 | Main interrupt stack size in bytes |
| `Interrupt_Secondary_Stack_Size` | Integer | ≥1 | 128 | Secondary interrupt stack size in bytes |

### Disabling PLL Q Output

If you don't need the 48 MHz clock for USB/SDIO/RNG:

```toml
[configuration.values]
light_tasking_stm32f746.PLL_Q_Enable = false
```

**Note:** The PLL will still be enabled if `SYSCLK_Src = "PLLCLK"`.

## Interrupt Stack Configuration

Customize interrupt stack sizes based on your application's needs:

```toml
[configuration.values]
light_tasking_stm32f746.Interrupt_Stack_Size = 2048
light_tasking_stm32f746.Interrupt_Secondary_Stack_Size = 256
```

## Project Files

The runtime provides two project files:

- **[`runtime_build.gpr`](runtime_build.gpr)** - Main runtime build project (recommended)
- **[`ravenscar_build.gpr`](ravenscar_build.gpr)** - Alternative Ravenscar-compatible build

## Directory Structure

```
light-tasking-stm32f746/
├── adalib/              # Compiled runtime library
├── gnat/                # GNAT runtime sources
├── gnat_user/           # User-configurable GNAT sources
├── gnarl/               # GNAT runtime library sources
├── gnarl_user/          # User-configurable GNARL sources
├── ld/                  # Linker scripts
├── ld_user/             # User-configurable linker scripts
├── obj/                 # Build artifacts
├── alire/               # Alire metadata
├── runtime_build.gpr    # Main runtime project file
├── ravenscar_build.gpr  # Ravenscar runtime project file
├── target_options.gpr   # Target-specific options
└── runtime.xml          # Runtime configuration
```

## Authors and Maintainers

- **Authors:** AdaCore, Daniel King, Olivier Henley
- **Maintainer:** Olivier Henley <olivier.henley@gmail.com>

## License

This runtime is licensed under GPL-3.0-or-later WITH GCC-exception-3.1.

## Resources

- **Website:** https://github.com/GNAT-Academic-Program/stm32f746-runtimes
- **STM32F746 Reference Manual:** RM0385
- **Tags:** embedded, runtime, stm32f7, stm32f746

## Dependencies

- **gnat_arm_elf:** ^15

## Troubleshooting

### Compile-Time PLL Errors

The runtime performs compile-time validation of PLL configurations. If you receive an error:

1. Verify VCO input frequency is between 1-2 MHz: `source_freq / PLL_M_Div`
2. Verify VCO output frequency is between 100-432 MHz: `(source_freq / PLL_M_Div) × PLL_N_Mul`
3. Check that your HSE frequency matches your hardware
4. Ensure APB1 frequency doesn't exceed 54 MHz
5. Ensure APB2 frequency doesn't exceed 108 MHz

### Example Valid Configurations

**HSI 16 MHz → 216 MHz SYSCLK:**
- M=16, N=432, P=DIV2 → VCO_in=1MHz, VCO_out=432MHz, SYSCLK=216MHz

**HSE 8 MHz → 216 MHz SYSCLK:**
- M=8, N=432, P=DIV2 → VCO_in=1MHz, VCO_out=432MHz, SYSCLK=216MHz

**HSE 25 MHz → 200 MHz SYSCLK:**
- M=25, N=400, P=DIV2 → VCO_in=1MHz, VCO_out=400MHz, SYSCLK=200MHz
