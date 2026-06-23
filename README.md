> **Fork note (Zephyr 4.x support):** This is a fork of
> [kumamuk-git/zmk-pmw3610-driver](https://github.com/kumamuk-git/zmk-pmw3610-driver).
> ZMK on **Zephyr 4.1** ships a built-in `pixart,pmw3610` input binding, which collides with this
> driver's same-named binding (`devicetree error: ... duplicate compatible on-bus spi`).
> To let this driver's driver-level discrete scrolling (`CONFIG_PMW3610_SCROLL_TICK`, `scroll-layers`)
> build on current ZMK/Zephyr 4.x, the **compatible is renamed to `pixart,pmw3610-kuma`**
> (binding + `DT_DRV_COMPAT`). Kconfig symbols (`CONFIG_PMW3610*`) are unchanged. Use
> `compatible = "pixart,pmw3610-kuma";` on the trackball node (see example below).

PMW3610 driver implementation for ZMK with at least Zephyr 3.5

This work is based on [ufan's implementation](https://github.com/ufan/zmk/tree/support-trackpad) of the driver.

## Installation

Only GitHub actions builds are covered here. Local builds are different for each user, therefore it's not possible to cover all cases.

Include this project on your ZMK's west manifest in `config/west.yml`:

```yml
manifest:
  remotes:
    - name: zmkfirmware
      url-base: https://github.com/petejohanson
    - name: inorichi
      url-base: https://github.com/inorichi
  projects:
    - name: zmk
      remote: zmkfirmware
      revision: feat/pointers-move-scroll
      import: app/west.yml
    - name: zmk-pmw3610-driver
      remote: inorichi
      revision: main
  self:
    path: config
```

Then, edit your `build.yml` to look like this, 3.5 is now on main:

```yml
on: [workflow_dispatch]

jobs:
  build:
    uses: zmkfirmware/zmk/.github/workflows/build-user-config.yml@main
```

Now, update your `board.overlay` adding the necessary bits (update the pins for your board accordingly):

```dts
&pinctrl {
    spi0_default: spi0_default {
        group1 {
            psels = <NRF_PSEL(SPIM_SCK, 0, 8)>,
                <NRF_PSEL(SPIM_MOSI, 0, 17)>,
                <NRF_PSEL(SPIM_MISO, 0, 17)>;
        };
    };

    spi0_sleep: spi0_sleep {
        group1 {
            psels = <NRF_PSEL(SPIM_SCK, 0, 8)>,
                <NRF_PSEL(SPIM_MOSI, 0, 17)>,
                <NRF_PSEL(SPIM_MISO, 0, 17)>;
            low-power-enable;
        };
    };
};

&spi0 {
    status = "okay";
    compatible = "nordic,nrf-spim";
    pinctrl-0 = <&spi0_default>;
    pinctrl-1 = <&spi0_sleep>;
    pinctrl-names = "default", "sleep";
    cs-gpios = <&gpio0 20 GPIO_ACTIVE_LOW>;

    trackball: trackball@0 {
        status = "okay";
        compatible = "pixart,pmw3610-kuma";
        reg = <0>;
        spi-max-frequency = <2000000>;
        irq-gpios = <&gpio0 6 (GPIO_ACTIVE_LOW | GPIO_PULL_UP)>;

        /*   optional features   */
        // snipe-layers = <1>;
        // scroll-layers = <2 3>;
        // automouse-layer = <4>;

        /*   optional: ball action on specific layers  */
        // arrows {
        //     layers = <3>;
        //     bindings =
        //         <&kp RIGHT_ARROW>,
        //         <&kp LEFT_ARROW>,
        //         <&kp UP_ARROW>,
        //         <&kp DOWN_ARROW>;
        //
        //     /*   optional: ball action configuration  */
        //     tick = <10>;
        //     // wait-ms = <5>;
        //     // tap-ms = <5>;
        // };
    };
};

/ {
  trackball_listener {
    compatible = "zmk,input-listener";
    device = <&trackball>;

  };
};
```

Now enable the driver config in your `board.config` file (read the Kconfig file to find out all possible options):

```conf
CONFIG_SPI=y
CONFIG_INPUT=y
CONFIG_ZMK_MOUSE=y
CONFIG_PMW3610=y
```
