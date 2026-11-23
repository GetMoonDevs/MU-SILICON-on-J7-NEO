# Galaxy J7 Neo (SM-J701F) USB Host Porting

## Target hardware

- Device: Samsung Galaxy J7 Neo
- Model: SM-J701F
- Codename: j7velte
- SoC: Exynos 7870
- USB controller: Synopsys DesignWare USB 2.0 OTG, DWC2-compatible host core
- USB speed: High Speed, 480 Mbps maximum

## Correct controller identification

The SoC DTS declares the wrapper at `0x13600000` with compatible `samsung,exynos7870-dwusb2`. Its child node is labelled `synopsys,dwc3`, but that child label alone does not identify the actual Exynos 7870 host-controller programming model.

The matching Linux glue driver, `drivers/usb/dwc3/dwc3-exynos.c`, maps `samsung,exynos7870-dwusb2` to `dwc2_exynos7870`. It requests only the `usbdrd20` clock for this controller. This is the important distinction: J701F USB input must be brought up as an Exynos DWC2 host, not by blindly attaching the generic PCI xHCI driver.

Verified register map from the J7 Neo device tree:

- USB controller base: `0x13600000`
- Controller window: `0x10000`
- Controller IRQ: 230
- PHY base: `0x135c0000`
- PHY window: `0x100`
- PHY compatible: `samsung,exynos7870-usbdrd-phy`
- PHY clocks: `usbdrd20`, `phyumux`, `usb_pll`
- Clock IDs in the kernel DTS: 335, 336, 350
- PMU syscon base: `0x10480000`
- PMU USB PHY control offset: `0x0704`
- Exynos 7870 USB2 PHY enable bit: bit 1
- Board property: `is_not_vbus_pad`

The board DTS marks the controller OTG and limits it to high-speed USB. The host driver must therefore use the USB 2.0 UTMI path. Do not enable SuperSpeed or assume that the PHY drives VBUS.

## Verified Linux PHY sequence

The Exynos 7870 Linux PHY driver provides the device-specific sequence. It:

1. Enables the USB2 PHY through PMU register `0x10480704`, bit 1.
2. Programs PHY clock/reset register `0x135c0010`, selecting PADREFCLK and the board reference clock.
3. Enables UTMI suspend/common-on and pulses the PHY port reset.
4. Applies the Exynos 7870 PLL tuning at PHY offset `0x70`.
5. Releases SIDDQ and PHY reset at offset `0x54`.
6. Sets forced BVALID/VBUSVALID because the board does not use the PHY VBUS pad.
7. Configures UTMI VBUS-valid external selection and disables the OTG comparator.
8. Applies the Exynos 7870 UTMI post-initialization tuning values.

The reference clock selection must match the real board clock. Do not hardcode a guessed frequency. The Linux driver resolves this from the clock provider and PHY configuration.

The DWC2 glue driver also enables the `usbdrd20` clock. The remaining PHY clocks must be enabled through the Exynos clock provider or a verified equivalent. Their gate registers must not be guessed.

## UEFI driver requirements

The required UEFI host controller is a DWC2 host-controller driver that publishes `EFI_USB2_HC_PROTOCOL`. The existing EDK2 Raspberry Pi `DwUsbHostDxe` is a useful reference for the transfer engine and USB2 HC protocol, but its BCM2836 base address, Raspberry Pi firmware power protocol, and platform assumptions cannot be reused unchanged.

After the host controller publishes `EFI_USB2_HC_PROTOCOL`, connect:

- `UsbBusDxe`
- `UsbKbDxe`
- USB mouse HID driver
- `UsbMassStorageDxe`

The generic PCI `XhciDxe` is not the correct driver for this Exynos DWC2 controller.

## Work completed

- [x] Confirmed SM-J701F / j7velte / Exynos 7870.
- [x] Located the exact J7 Neo USB controller and PHY nodes.
- [x] Verified controller and PHY MMIO bases.
- [x] Read the Exynos Linux glue driver and confirmed the DWC2 host-controller path.
- [x] Read the Exynos 7870 PHY initialization sequence and PMU enable bit.
- [x] Identified that the generic PCI xHCI driver is the wrong controller driver.
- [ ] Port a DWC2 host-controller DXE driver and remove Raspberry Pi platform dependencies.
- [ ] Implement verified Exynos 7870 clock and PHY initialization in UEFI.
- [ ] Integrate and start UEFI USB bus, keyboard, mouse, and mass-storage drivers.
- [ ] Build and validate on the physical J701F with a powered OTG hub, keyboard, mouse, and flash drive.

## Source references

- J7 Neo board DTS: https://github.com/Exynos7870/android_kernel_samsung_universal7870/blob/lineage-16.0/arch/arm64/boot/dts/exynos7870-j7velte_common.dtsi
- Exynos 7870 SoC DTS: https://github.com/Exynos7870/android_kernel_samsung_exynos7870/blob/lineage-16.0/arch/arm64/boot/dts/exynos7870.dtsi
- Exynos USB glue driver: https://github.com/Exynos7870/android_kernel_samsung_exynos7870/blob/lineage-16.0/drivers/usb/dwc3/dwc3-exynos.c
- Exynos 7870 PHY driver: https://github.com/torvalds/linux/blob/master/drivers/phy/samsung/phy-exynos5-usbdrd.c
- Exynos 7870 PMU definitions: https://github.com/raspberrypi/linux/blob/master/include/linux/soc/samsung/exynos-regs-pmu.h
- EDK2 DWC2 host-controller reference: https://github.com/tianocore/edk2-platforms/tree/master/Platform/RaspberryPi/Drivers/DwUsbHostDxe

Do not flash an experimental image over the only working UEFI build. Keep the known-working image and test a separate artifact.
