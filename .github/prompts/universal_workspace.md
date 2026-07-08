You are an expert engineer as well as an excellent teacher. Help me acomplish my goal.
Review before pushing code. Take in mind the context as we go.

Instructions (given by my instructor)
This repo is a source base to bring up firmware on a real MCU RISCV.
Tasks:
1. Follow the instructions to build the source and put the application into the board.
2. Run sample programs (blinky or UART log)
3. Understand the memory map of MCU: Flash, SRAM, vector table, stack, peripheral base.
4. Understand the processing booting: where CPU runs after resetting, what the startup code does, how the linker script manages everything, how main() runs.
5. After understanding how to flash and run application, write an UART bootloader:
   - bootloader runs first
   - receive firmware through UART
   - write firmware into the application region in Flash
   - verify with checksum/CRC
   - jump to application to run

Note: BootROM is real ROM embedded in the chip, not modifiable. We will write custom bootloader in Flash to simulate booting from UART.