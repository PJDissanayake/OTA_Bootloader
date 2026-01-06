# STM32F407 OTA Bootloader Project

**Author**: Pathum J Dissanayake  
**Created**: January 06, 2026  
**Target MCU**: STM32F407VG/T (1MB Flash, 192KB RAM total - 128KB SRAM + 64KB CCMRAM)  

This is a **secure custom bootloader** implementing the XCP protocol over SPI2 slave for Over-The-Air (OTA) firmware updates. It includes persistent update requests from the application and a safe jump mechanism to the main firmware.

## Features

- **XCP protocol** over SPI2 slave for firmware flashing (CONNECT, PROGRAM, etc.)
- **Persistent OTA request** using RTC backup register (survives reset/power cycle)
- **Safe jump** to application with full peripheral cleanup (SPI DMA stop, deinit, IRQ disable)
- **Flash partitioning**:
  - Bootloader: 32KB at `0x08000000`
  - Application: 992KB at `0x08008000`
- **Application validity check** (stack pointer and reset handler)
- **LED feedback**:
  - PA6: Bootloader status / recovery mode
- **SPI2 pins** (slave mode): PB12 (NSS), PB13 (SCK), PB14 (MISO), PB15 (MOSI)

## Memory Layout

| Region               | Address            | Size   | Description                          |
|----------------------|--------------------|--------|--------------------------------------|
| Bootloader           | 0x08000000         | 32KB   | Bootloader code & vectors            |
| Application          | 0x08008000         | 992KB  | Main firmware (code + relocated vectors) |
| Main SRAM            | 0x20000000         | 128KB  | Stack at top (0x20020000)            |
| CCMRAM               | 0x10000000         | 64KB   | Optional core-coupled RAM            |

## How It Works

1. **On Reset**:
   - Bootloader starts
   - Enables PWR clock and backup access
   - Checks RTC BKP0R for `0xDEADBEEF` (OTA request)
   - If found → clears flag → forces OTA mode (`xcp_connected = true`)

2. **Normal Boot**:
   - Waits 3 seconds for XCP CONNECT on SPI2
   - If received → PA6 ON → stays in OTA mode
   - Else → checks application validity
   - If valid → cleans SPI2/DMA → jumps to app

3. **Recovery**:
   - Invalid app → fast blink PA6

## Flashing Instructions

### Using STM32CubeIDE

- **Bootloader**: Run normally (no offset) → flashes to `0x08000000`
- **Application**:
  - Run Configurations → Startup tab
  - Check **"Use download offset"**
  - Offset: `0x8000`
  - Run → flashes to `0x08008000`

### Using STM32CubeProgrammer

- Bootloader: Address `0x08000000`
- Application: Address `0x08008000`

## Trigger OTA from Application

Add this function in your **application** code:

```c
void RequestOTAUpdate(void)
{
  __HAL_RCC_PWR_CLK_ENABLE();
  HAL_PWR_EnableBkUpAccess();
  RTC->BKP0R = 0xDEADBEEF;
  NVIC_SystemReset();
}
```

Call when firmware update is needed (e.g., button press, received command).

## Debugging Tips

- PA6 fast blink → invalid application (wrong stack or reset handler)
- PA6 3 long blinks (in debug build) → jump successful
- Use STM32CubeProgrammer to verify memory at `0x08008000`:
  - First word should be `0x20020000` (stack)
  - Second word ~`0x08009xxx` (reset handler)

## License

Provided AS-IS. Use at your own risk.

---

**Enjoy your secure OTA system!** 🚀

For questions or improvements, contact the author.
