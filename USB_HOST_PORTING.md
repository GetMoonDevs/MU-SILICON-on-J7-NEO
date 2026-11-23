# Galaxy J7 Neo (SM-J701F) USB Host Porting

## Confirmed platform

- Target: Samsung Galaxy J7 Neo, SM-J701F.
- SoC: Samsung Exynos 7870 (S5E7870).
- The upstream Exynos 7870 device-tree description places the USB DRD glue/controller at `0x13600000` with a `0x10000` register window.
- The USB2 DRD PHY is described at `0x135c0000`.
- The upstream device tree associates the controller with three FSYS clocks named `bus_early`, `ref`, and `ctrl`, and a PHY clock/reference pair.
- The current Mu-Silicium `UsbfnDwc3Dxe` module implements the USB Function (device/gadget) protocol. It is not a USB Host Controller driver.

## Important distinction

Adding `UsbfnDwc3Dxe.inf` to the J7 Neo platform will not make a USB keyboard, mouse, or flash drive work as a host. That driver handles the phone acting as a USB device.

USB Host support needs two separate layers:

1. A platform bring-up driver that enables the Exynos 7870 FSYS clocks, initializes/releases the USB PHY, resets the DWC3 core, and selects host mode.
2. A host-controller driver that exposes the DWC3 host-side xHCI register interface through UEFI's USB Host Controller protocol. The standard USB bus, HID, and mass-storage drivers can then enumerate peripherals.

## Register and platform references

- DWC3 glue/controller base: `0x13600000`.
- USB DRD PHY base: `0x135c0000`.
- Linux upstream compatible: `samsung,exynos7870-dwusb3`.
- DWC3 core compatible: `snps,dwc3`.
- The DWC3 global control register's port-capability field must select host mode. This must happen only after the PHY and required clocks are ready.

## Current implementation status

- [x] Confirm target model and Exynos 7870 family.
- [x] Identify the upstream DWC3 and PHY register windows.
- [x] Confirm the existing Mu-Silicium DWC3 function driver is not a host driver.
- [ ] Confirm the SM-J701F board-specific PHY, clock, and VBUS/OTG sequencing from its exact kernel device tree or vendor sources.
- [ ] Implement and validate the Exynos 7870 DWC3 host bring-up DXE.
- [ ] Integrate a compatible xHCI host-controller driver.
- [ ] Add UEFI USB bus, keyboard/mouse HID, and mass-storage drivers.
- [ ] Validate enumeration with a powered OTG hub, USB keyboard/mouse, and flash drive.

## Safety

Do not flash an experimental host driver over the only working UEFI image. Keep the known-working build and test the new image from a separate artifact first.
