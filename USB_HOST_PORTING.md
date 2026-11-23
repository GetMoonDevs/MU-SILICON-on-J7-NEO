# Galaxy J7 Neo (SM-J701F) USB Host Porting

## Confirmed target hardware

Target: Samsung Galaxy J7 Neo, model SM-J701F, codename j7velte, Exynos 7870.

The exact J7 Neo Linux device tree is available in:
`Exynos7870/android_kernel_samsung_universal7870`, branch `lineage-16.0`.

Relevant files:
- `arch/arm64/boot/dts/exynos7870-j7velte_common.dtsi`
- `arch/arm64/boot/dts/exynos7870.dtsi`

## Verified USB hardware configuration

The J7velte board file enables the USB controller and configures its DWC3 child as follows:

- Controller node: `usb@13600000`
- Controller register base: `0x13600000`
- Register window: `0x10000`
- DWC3 core compatible: `synopsys,dwc3`
- USB maximum speed: `high-speed` (USB 2.0)
- Linux role mode: `otg`
- Board property: `is_not_vbus_pad`

The Exynos 7870 SoC device tree defines the DRD PHY:

- PHY base: `0x135c0000`
- PHY register window: `0x100`
- PHY compatible: `samsung,exynos7870-usbdrd-phy`
- Clocks: `usbdrd20`, `phyumux`, `usb_pll`
- Clock IDs in this kernel tree: 335, 336, 350
- PMU syscon: `pmu_system_controller`

The DWC3 node uses IRQ 230 and references USB2 and USB3 PHY indices. The board limits the controller to high-speed USB, so the first target is USB 2.0 OTG host mode.

## Existing Mu-Silicium limitation

Mu-Silicium contains `SamsungPkg/Drivers/UsbfnDwc3Dxe`. This implements the USB Function protocol for device/gadget mode. It is not a USB Host Controller driver.

The current Mu-Silicium source tree does not contain an obvious DWC3/xHCI host DXE driver. The pinned Mu_Basecore submodule does contain the generic EDK2 XhciDxe, UsbBusDxe, UsbKbDxe, and UsbMassStorageDxe modules. Mu_Plus contains UsbMouseHidDxe. However, XhciDxe lives under Bus/Pci and uses PCI services. The J7 Neo DWC3 is an SoC MMIO controller, not a PCI device, so it needs a platform adaptation or a dedicated host-controller DXE that publishes EFI_USB2_HC_PROTOCOL. The generic USB bus and class drivers can then sit above that protocol.

## Required implementation

1. Implement Exynos 7870 USB PHY and clock bring-up using the verified J7 Neo register map and board properties.
2. Reset and initialize the DWC3 core, then select host port capability.
3. Expose the DWC3 host-side xHCI register interface through UEFI's USB Host Controller protocol.
4. Connect the UEFI USB bus driver, HID keyboard/mouse drivers, and USB mass-storage driver.
5. Verify VBUS/OTG power behavior. The board's `is_not_vbus_pad` property means VBUS handling must not be guessed from a generic Exynos board.
6. Test with a powered OTG hub, keyboard, mouse, and USB flash drive.

## Status

- [x] Confirm SM-J701F / j7velte / Exynos 7870.
- [x] Read the exact J7velte board USB node.
- [x] Verify DWC3 base, PHY base, IRQ, clock names, and high-speed OTG configuration.
- [x] Confirm the existing Mu-Silicium DWC3 driver is device mode, not host mode.
- [x] Locate the generic EDK2 USB stack in Mu_Basecore: XhciDxe, UsbBusDxe, UsbKbDxe, UsbMassStorageDxe. Mu_Plus also contains USB mouse HID support.\n- [ ] Adapt or implement a platform xHCI host-controller driver for the DWC3 MMIO controller. The generic XhciDxe is under Bus/Pci and expects PCI services, so it cannot be assumed to bind directly to this non-PCI SoC controller.
- [ ] Implement Exynos 7870 PHY/clock/PMU initialization.
- [ ] Integrate USB bus, HID, and mass-storage drivers.
- [ ] Build and test on the physical J7 Neo.

## References

- J7 Neo board DTS: https://github.com/Exynos7870/android_kernel_samsung_universal7870/blob/lineage-16.0/arch/arm64/boot/dts/exynos7870-j7velte_common.dtsi
- Exynos 7870 SoC DTS: https://github.com/Exynos7870/android_kernel_samsung_universal7870/blob/lineage-16.0/arch/arm64/boot/dts/exynos7870.dtsi
- Mu-Silicium USB Function driver: https://github.com/Project-Silicium/Mu-Silicium/tree/main/Silicon/Samsung/SamsungPkg/Drivers/UsbfnDwc3Dxe

Do not flash an experimental image over the only working UEFI build. Keep the known-working image and test the new build as a separate artifact.
