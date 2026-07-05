## D-Link DIR-300 B3 WAN led fix

For revision **B3**, vendor changed the GPIO wiring for the LEDs compared to revisions B1/B2. Because of this, after flashing with the original padavan-ng, WAN led does not work, and the Power LED will stay always amber instead of green after boot.

I was able to find correct gpio for B3 board

#### Power led:
```
- mtk_gpio -w 9 1 - POWER led OFF
- mtk_gpio -w 9 0 - POWER led on GREEN
- mtk_gpio -w 8 1 - POWER led COLOR GREEN
- mtk_gpio -w 8 0 - POWER led COLOR AMBER
```

#### WAN led:
```
- mtk_gpio -w 12 1 - WAN led OFF
- mtk_gpio -w 12 0 - WAN led on GREEN
- mtk_gpio -w 14 1 - WAN led COLOR GREEN
- mtk_gpio -w 14 0 - WAN led COLOR ANBER
```

### Solution:
Add support for MediaTek low-level utilities to the build config:
```
- CONFIG_FIRMWARE_INCLUDE_GPIO=y
- CONFIG_FIRMWARE_INCLUDE_GPIO_UTILS=y
```

Remove any additional locales to preserve flash space (original config build will fail while include any one of them)

Or use [`dir300_b3_wan-led-fix_build.config`](../dir300_b3_wan-led-fix_build.config) as reference for github build steps below, build firmware and install it to your device.

Enable ssh access via webui (Advanced Settings -> Services -> Enable SSH server), login via ssh and check mtk_gpio is in place, you may use any of led commands above. If you see it works then proceed with (Advanced Settings -> Scripts -> Run After Router Started) and add next:

```
# Make power led green after boot
mtk_gpio -w 8 1

# Enable WAN LED and set it's color ORANGE after bootup
mtk_gpio -w 12 0
mtk_gpio -w 14 0

# Check ISP connection and set WAN led green if internet is alive
(
while true; do
    if ping -c 1 -W 2 1.1.1.1 > /dev/null 2>&1; then
        # internet ok -> make wan led green
        mtk_gpio -w 14 1
    else
        # internet issue -> make wan led orange
        mtk_gpio -w 14 0
    fi
    sleep 10
done
) &
```

These steps will make the LED behavior the same as in the vendor firmware, traffic blinking also works
- GREEN power led after system bootup
- GREEN means internet connection is ok
- AMBER - there are issues with internet access



## Automatic Padavan firmware builds using GitHub servers

### Usage

- [Fork this repository](https://github.com/shvchk/padavan-builder-workflow/fork), further steps should be performed in your fork

- Copy your build config to [`build.config`](../build.config)

  Build config template can be found in the [firmware repository](https://gitlab.com/hadzhioglu/padavan-ng/-/tree/master/trunk/configs/templates)

- Run the build process: [Actions](../../../actions) → [Build firmware](../../../actions/workflows/build.yml) → Run workflow

  ![run workflow](misc/run-workflow.webp)

  The build process will appear on the same page (if it doesn't appear, just refresh the page). You can get process details by clicking on it.

  Depending on the build config, build process usually takes from 10 to 60 minutes.

- While the process is in progress, its status indicator would be gold-ish circle

  ![workflow status progress](misc/workflow-status-in-progress.webp)

- If the process finishes successfully, its status indicator would turn green with a check mark

  ![workflow status success](misc/workflow-status-success.webp)

  Click on the finished process. Archive with the firmware would be stored as its artifact:

  ![workflow artifacts](misc/workflow-artifacts.webp)

  Firmware license does not permit binaries distribution, so artifacts are stored for 7 days for personal use.

- If the process finishes with an error, its status indicator would turn red with a cross

  ![workflow status fail](misc/workflow-status-fail.webp)

  Click on the finished process. To get details about the error, click on the failed `build` job at the left:

  ![workflow details fail](misc/workflow-details-fail.webp)

  Job report will be opened:

  ![workflow details get logs](misc/workflow-details-get-logs.webp)

  Here it's immediately obvious that it was *Check firmware size* step that failed — it is marked with a red circle with a cross. Specific reason is shown below: *Firmware size (18,492,849 bytes) exceeds max size (16,187,392 bytes) for your target device* — i.e. built firmware size is too big for the target device.

  In case of any error its reason is usually shown at the end of the log, as in the example above. To view full log click on the cog ⚙️ icon in the top right corner → View raw logs. You can also download compressed log archive in the same menu → Download log archive.

  If you can't figure out the problem on your own, you can ask community or firmware developer for help. In this case don't forget to attach the log archive.


### Updating your fork

To sync your fork with its origin repository, just click *Sync fork* → *Update branch* at the top of the main page of your fork:

![sync fork](misc/sync-fork.webp)


### Advanced usage

You can set the firmware repository, branch, specific tag or commit in the [`variables`](../variables) file.

In the [`variables`](../variables) file you can also specify which themes you want to install by uncommenting theme names in the `PADAVAN_THEMES` variable. Themes repository can be set with the `PADAVAN_THEMES_REPO` variable.

You can create a `pre-build.sh` script with any custom commands, which will be executed just before build process. By that time firmware source code is already downloaded, so you can add or change anything in it.

You can create a `post-build.sh` script, which will be executed right after build process.


---

Discussion: https://github.com/shvchk/padavan-builder-workflow/discussions/categories/general
