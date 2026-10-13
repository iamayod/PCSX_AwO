# Vita change review guide

Seven focused implementation commits replace the original combined commit `2060bdcd`. The series is based on upstream `3dd519c4`, preserving its LiveArea launch, NEON synchronization, and VRAM compositing fixes.

Each table covers every added and removed line in its commit, including comments and structural lines. `+` line numbers refer to that commit; `-` line numbers refer to its parent. Later commits may move those lines. Whitespace is trimmed in the displayed code only.

Source comments explain behavior and intent. This guide also describes removed lines and syntax-only changes without filling production code with comments on braces.

Diagnostics retain the original behavior: GL error reads consume the error flag; skipped-presentation logging can repeat until eight submission attempts occur; renderer log streams persist for the process. Hardware verification is still required.

## build(vita): configure CTR headers and Python 3 tooling

Commit: `e155431c`

Make CTR_NATIVE_DIR configurable, prioritize the compatibility headers, expose the libchdr include directory, and invoke the shader embedding tool with Python 3. Comments explain header ordering and recursive argument propagation.

### [Makefile](../Makefile)

| Line | Changed code | Purpose |
| --- | --- | --- |
| -287 | <code>CFLAGS += -DGPU_VITA_CTR -DCTR_NATIVE -Iplugins/gpu_vita_ctr/compat -I../ctr-native/include</code> | Retain the GPU feature defines and compatibility include path; the configurable CTR path is handled separately. |
| +287 | <code># Allow another CTR checkout without editing the build rules.</code> | Source comment explaining the adjacent change: Allow another CTR checkout without editing the build rules. |
| +288 | <code>CTR_NATIVE_DIR ?= ../Crash-Team-Racing-High-Octane</code> | Supply an overridable default for the CTR header checkout. |
| +289 | <code># Resolve PCSX packed-vertex headers before SDK or CTR alternatives.</code> | Source comment explaining the adjacent change: Resolve PCSX packed-vertex headers before SDK or CTR alternatives. |
| +290 | <code>CFLAGS := -Iplugins/gpu_vita_ctr/compat -I$(CTR_NATIVE_DIR)/include $(CFLAGS)</code> | Prepend PCSX compatibility headers, then CTR headers, before existing include directories. |
| +291 | <code>CFLAGS += -DGPU_VITA_CTR -DCTR_NATIVE -Iplugins/gpu_vita_ctr/compat</code> | Retain the GPU feature defines and compatibility include path; the configurable CTR path is handled separately. |

### [Makefile.libretro](../Makefile.libretro)

| Line | Changed code | Purpose |
| --- | --- | --- |
| -336 | <code>CFLAGS += -I$(VITAGL_DIR)/source -I$(VITASDK)/include -Ifrontend/vita</code> | Supply Vita include directories; the added libchdr directory resolves unprefixed CHD header includes. |
| +336 | <code># Expose libchdr headers included without the libchdr/ prefix on Vita.</code> | Source comment explaining the adjacent change: Expose libchdr headers included without the libchdr/ prefix on Vita. |
| +337 | <code>CFLAGS += -I$(VITAGL_DIR)/source -I$(VITASDK)/include -Ifrontend/vita -Ideps/libchdr/include/libchdr</code> | Supply Vita include directories; the added libchdr directory resolves unprefixed CHD header includes. |

### [Makefile.vita](../Makefile.vita)

| Line | Changed code | Purpose |
| --- | --- | --- |
| +8 | <code>endif</code> | End the conditional default assignment. |
| +9 | <code># Use the Vita High Octane checkout unless the caller supplies another location.</code> | Source comment explaining the adjacent change: Use the Vita High Octane checkout unless the caller supplies another location. |
| +10 | <code>ifeq ($(strip $(CTR_NATIVE_DIR)),)</code> | Use the default only when the supplied CTR path is empty. |
| +11 | <code>CTR_NATIVE_DIR := ../Crash-Team-Racing-High-Octane</code> | Choose the sibling High Octane checkout as the default header source. |
| +48 | <code># Forward the same header source to the recursive core build.</code> | Source comment explaining the adjacent change: Forward the same header source to the recursive core build. |
| +49 | <code>CORE_ARGS += CTR_NATIVE_DIR=$(CTR_NATIVE_DIR)</code> | Pass the selected CTR path into the recursive core build. |
| +50 | <code>(blank)</code> | Visual separation only; no runtime effect. |
| -78 | <code>python tools/embed_vita_postfx.py $(BUILD_DIR)/postfx $@ $(POSTFX_NAMES)</code> | Replace the ambiguous python executable with an explicit Python 3 invocation. |
| +85 | <code># The embedding script requires Python 3, independently of the python alias.</code> | Source comment explaining the adjacent change: The embedding script requires Python 3, independently of the python alias. |
| +86 | <code>python3 tools/embed_vita_postfx.py $(BUILD_DIR)/postfx $@ $(POSTFX_NAMES)</code> | Run the shader embedding script with Python 3 and preserve its arguments. |

## fix(vita): enforce vertex ABI and initialize SDL video

Commit: `2a96801d`

Assert the packed GrVertex size in all three GPU translation units, run the core GPU synchronously, and initialize/quit the SDL video subsystem around emulation. Fail startup cleanly if video initialization fails.

### [Makefile.vita](../Makefile.vita)

| Line | Changed code | Purpose |
| --- | --- | --- |
| +45 | <code># Keep GPU submission on the calling thread for the Vita renderer.</code> | Source comment explaining the adjacent change: Keep GPU submission on the calling thread for the Vita renderer. |
| -46 | <code>USE_ASYNC_CDROM=1 USE_ASYNC_GPU=1 USE_ASYNC_SPU=0 VITAGL_DIR=$(VITAGL_DIR)</code> | Keep CD-ROM/SPU choices and vitaGL path; GPU=0 disables the asynchronous GPU worker, while GPU=1 is the prior setting. |
| +47 | <code>USE_ASYNC_CDROM=1 USE_ASYNC_GPU=0 USE_ASYNC_SPU=0 VITAGL_DIR=$(VITAGL_DIR)</code> | Keep CD-ROM/SPU choices and vitaGL path; GPU=0 disables the asynchronous GPU worker, while GPU=1 is the prior setting. |

### [frontend/vita/standalone.c](../frontend/vita/standalone.c)

| Line | Changed code | Purpose |
| --- | --- | --- |
| +660 | <code>/* Initialize SDL video before the renderer creates its window/context. */</code> | Source comment explaining the adjacent change: Initialize SDL video before the renderer creates its window/context. |
| +661 | <code>if (!SDL_InitSubSystem(SDL_INIT_VIDEO)) {</code> | Initialize SDL video and branch to cleanup on failure. |
| +662 | <code>sceClibPrintf(&quot;PCSX AwO: SDL video init failed: %s\n&quot;, SDL_GetError());</code> | Print SDL diagnostic text when video startup fails. |
| +663 | <code>display_finish();</code> | Release the already initialized display before returning from failed startup. |
| +664 | <code>return 1;</code> | Exit with failure when SDL video cannot be initialized. |
| +665 | <code>}</code> | Open or close the surrounding block/declaration; no independent operation. |
| +666 | <code>sceClibPrintf(&quot;PCSX AwO: video initialized\n&quot;);</code> | Report successful video startup to the platform console. |
| +667 | <code>(blank)</code> | Visual separation only; no runtime effect. |
| +849 | <code>/* Release SDL video after the emulation core has shut its renderer down. */</code> | Source comment explaining the adjacent change: Release SDL video after the emulation core has shut its renderer down. |
| +850 | <code>SDL_QuitSubSystem(SDL_INIT_VIDEO);</code> | Shut down SDL video after core/renderer teardown. |

### [plugins/gpu_vita_ctr/gpulib_if.c](../plugins/gpu_vita_ctr/gpulib_if.c)

| Line | Changed code | Purpose |
| --- | --- | --- |
| +19 | <code>/* Reject the CTR/PGXP layout: PS1 vertex attributes require a packed 22-byte stride. */</code> | Source comment explaining the adjacent change: Reject the CTR/PGXP layout: PS1 vertex attributes require a packed 22-byte stride. |
| +20 | <code>_Static_assert(sizeof(GrVertex) == 22, &quot;PCSX requires its compatibility vertex header&quot;);</code> | Fail compilation if a conflicting header changes the packed PS1 vertex stride from 22 bytes. |

### [plugins/gpu_vita_ctr/native_gpu.c](../plugins/gpu_vita_ctr/native_gpu.c)

| Line | Changed code | Purpose |
| --- | --- | --- |
| +11 | <code>/* Reject the CTR/PGXP layout: PS1 vertex attributes require a packed 22-byte stride. */</code> | Source comment explaining the adjacent change: Reject the CTR/PGXP layout: PS1 vertex attributes require a packed 22-byte stride. |
| +12 | <code>_Static_assert(sizeof(GrVertex) == 22, &quot;PCSX GPU must emit packed PS1 vertices&quot;);</code> | Fail compilation if a conflicting header changes the packed PS1 vertex stride from 22 bytes. |

### [plugins/gpu_vita_ctr/native_renderer.c](../plugins/gpu_vita_ctr/native_renderer.c)

| Line | Changed code | Purpose |
| --- | --- | --- |
| +6 | <code>/* Reject the CTR/PGXP layout: PS1 vertex attributes require a packed 22-byte stride. */</code> | Source comment explaining the adjacent change: Reject the CTR/PGXP layout: PS1 vertex attributes require a packed 22-byte stride. |
| +7 | <code>_Static_assert(sizeof(GrVertex) == 22, &quot;PCSX requires its compatibility vertex header&quot;);</code> | Fail compilation if a conflicting header changes the packed PS1 vertex stride from 22 bytes. |

## feat(vita): add DualShock device and analog mode settings

Commit: `e296ee87`

Expose Standard/DualShock and analog/digital settings, export an explicit core pad setter, and enable extended analog sampling. Apply device selection after game load/restart and apply mode changes on resume without overriding game-selected modes on every poll. Disable the legacy analog shortcut.

### [Makefile.libretro](../Makefile.libretro)

| Line | Changed code | Purpose |
| --- | --- | --- |
| +352 | <code># Keep the menu-controlled analog setter visible to the standalone executable.</code> | Source comment explaining the adjacent change: Keep the menu-controlled analog setter visible to the standalone executable. |
| +353 | <code>EXTRA_EXTERN_SYMS += padSetAnalog</code> | Expose the core analog setter to the separately linked standalone frontend. |

### [frontend/vita/rom_selector.cpp](../frontend/vita/rom_selector.cpp)

| Line | Changed code | Purpose |
| --- | --- | --- |
| +192 | <code>/* Stored values match the standalone frontend; labels describe the device/mode separately. */</code> | Source comment explaining the adjacent change: Stored values match the standalone frontend; labels describe the device/mode separately. |
| +193 | <code>static const OptionChoice k_controller_choices[] = {</code> | Define the two controller-device choices shown in the menu. |
| +194 | <code>{ &quot;standard&quot;, &quot;Standard (Digital)&quot; },</code> | Map the persisted standard value to the digital-controller label. |
| +195 | <code>{ &quot;dualshock&quot;, &quot;DualShock&quot; }</code> | Map the persisted dualshock value to the DualShock label. |
| +196 | <code>};</code> | Open or close the surrounding block/declaration; no independent operation. |
| +197 | <code>static const OptionChoice k_controller_mode_choices[] = {</code> | Define the independent analog/digital mode choices. |
| +198 | <code>{ &quot;digital&quot;, &quot;Digital&quot; },</code> | Map the digital mode value to its menu label. |
| +199 | <code>{ &quot;analog&quot;, &quot;Analog&quot; }</code> | Map the analog mode value to its menu label. |
| +200 | <code>};</code> | Open or close the surrounding block/declaration; no independent operation. |
| +201 | <code>(blank)</code> | Visual separation only; no runtime effect. |
| +203 | <code>{ &quot;SYSTEM&quot;, &quot;DualShock Mode&quot;, &quot;Select Analog or Digital for the DualShock controller. Applied at game start or when changed in the pause menu. Requires Controller set to DualShock. Games can subsequently change the mode.&quot;, &quot;vita_controller_mode&quot;, k_controller_mode_choices, 2, false },</code> | Add mode selection and explain when it applies and that games may later change it. |
| +204 | <code>{ &quot;SYSTEM&quot;, &quot;Controller&quot;, &quot;Port 1 controller. DualShock sends both Vita sticks as real analog axes. Applied on the next game start/restart. Standard preserves digital-only compatibility.&quot;, &quot;vita_controller&quot;, k_controller_choices, 2, false },</code> | Add port-1 device selection and explain the required game restart. |

### [frontend/vita/settings.c](../frontend/vita/settings.c)

| Line | Changed code | Purpose |
| --- | --- | --- |
| +18 | <code>/* Preserve digital compatibility by default; analog is the initial DualShock mode. */</code> | Source comment explaining the adjacent change: Preserve digital compatibility by default; analog is the initial DualShock mode. |
| +19 | <code>{ &quot;vita_controller&quot;, &quot;standard&quot;, {0} },</code> | Persist controller type with Standard as the compatibility-preserving default. |
| +20 | <code>{ &quot;vita_controller_mode&quot;, &quot;analog&quot;, {0} },</code> | Persist the requested mode, defaulting to analog when DualShock is selected. |
| +122 | <code>/* Mode is controlled by the menu, never by a legacy saved shortcut. */</code> | Source comment explaining the adjacent change: Mode is controlled by the menu, never by a legacy saved shortcut. |
| +123 | <code>if (key &amp;&amp; strcmp(key, &quot;pcsx_rearmed_analog_combo&quot;) == 0)</code> | Intercept the legacy analog shortcut option while guarding a null key. |
| +124 | <code>return &quot;disabled&quot;;</code> | Disable the legacy shortcut so the menu controls user-requested mode changes. |

### [frontend/vita/standalone.c](../frontend/vita/standalone.c)

| Line | Changed code | Purpose |
| --- | --- | --- |
| +69 | <code>/* The standalone frontend links this exported core entry point directly. */</code> | Source comment explaining the adjacent change: The standalone frontend links this exported core entry point directly. |
| +70 | <code>extern int padSetAnalog(unsigned int index, int analog);</code> | Declare the core setter used by the standalone frontend. |
| +71 | <code>/* Remember the active device; editing the menu does not hot-plug a controller. */</code> | Source comment explaining the adjacent change: Remember the active device; editing the menu does not hot-plug a controller. |
| +72 | <code>static bool g_controller_dualshock;</code> | Cache the active device independently of pending menu edits. |
| +73 | <code>/* Remember the requested mode so unchanged menu visits preserve game-selected modes. */</code> | Source comment explaining the adjacent change: Remember the requested mode so unchanged menu visits preserve game-selected modes. |
| +74 | <code>static bool g_controller_analog;</code> | Cache the last requested mode to detect a user change on resume. |
| +75 | <code>(blank)</code> | Visual separation only; no runtime effect. |
| +76 | <code>static void apply_controller_settings(bool loaded)</code> | Centralize applying device/mode settings after load or on resume. |
| +77 | <code>{</code> | Open or close the surrounding block/declaration; no independent operation. |
| +78 | <code>/* Read the persisted port-1 device and the user-requested analog mode. */</code> | Source comment explaining the adjacent change: Read the persisted port-1 device and the user-requested analog mode. |
| +79 | <code>const char *controller = VitaSettings_GetOption(&quot;vita_controller&quot;);</code> | Read the persisted controller-device selection. |
| +80 | <code>const char *mode = VitaSettings_GetOption(&quot;vita_controller_mode&quot;);</code> | Read the persisted analog/digital mode selection. |
| +81 | <code>bool dualshock = controller &amp;&amp; strcmp(controller, &quot;dualshock&quot;) == 0;</code> | Treat only a present dualshock value as a DualShock request. |
| +82 | <code>bool analog = mode &amp;&amp; strcmp(mode, &quot;analog&quot;) == 0;</code> | Treat only a present analog value as an analog-mode request. |
| +83 | <code>/* Controller changes require a restart; mode changes apply on resume.</code> | Source comment explaining the adjacent change: Controller changes require a restart; mode changes apply on resume. |
| +84 | <code>* Do not override game-selected modes on every frame or menu visit. */</code> | Source comment explaining the adjacent change: Do not override game-selected modes on every frame or menu visit. |
| +85 | <code>if (loaded) {</code> | Change the emulated controller device only after a game load/restart. |
| +86 | <code>g_controller_dualshock = dualshock;</code> | Remember which controller device was actually selected for this game. |
| +87 | <code>retro_set_controller_port_device(0, dualshock ?</code> | Configure libretro port 0 from the selected controller type. |
| +88 | <code>RETRO_DEVICE_SUBCLASS(RETRO_DEVICE_ANALOG, 1) : RETRO_DEVICE_JOYPAD);</code> | Use the core DualShock subclass for analog or the standard joypad device otherwise. |
| +89 | <code>}</code> | Open or close the surrounding block/declaration; no independent operation. |
| +90 | <code>if (g_controller_dualshock &amp;&amp; (loaded &#124;&#124; analog != g_controller_analog))</code> | Apply a mode only to an active DualShock and only at load or after a changed request. |
| +91 | <code>padSetAnalog(0, analog);</code> | Set port 0 to the requested mode rather than blindly toggling it. |
| +92 | <code>/* Cache the request even for a digital pad; a restart reapplies it to DualShock. */</code> | Source comment explaining the adjacent change: Cache the request even for a digital pad; a restart reapplies it to DualShock. |
| +93 | <code>g_controller_analog = analog;</code> | Remember the request so unchanged menu visits preserve game-selected modes. |
| +94 | <code>}</code> | Open or close the surrounding block/declaration; no independent operation. |
| +678 | <code>/* Ext2 polling needs extended analog sampling to supply both stick axes. */</code> | Source comment explaining the adjacent change: Ext2 polling needs extended analog sampling to supply both stick axes. |
| +679 | <code>sceCtrlSetSamplingModeExt(SCE_CTRL_MODE_ANALOG_WIDE);</code> | Enable wide analog sampling for the extended controller API; diagnostics also record the return value. |
| +762 | <code>/* Loading resets controller types. Select afterward, before retro_run.</code> | Source comment explaining the adjacent change: Loading resets controller types. Select afterward, before retro_run. |
| +763 | <code>* Subclass matches RETRO_DEVICE_PSE_DUALSHOCK in the libretro core. */</code> | Source comment explaining the adjacent change: Subclass matches RETRO_DEVICE_PSE_DUALSHOCK in the libretro core. |
| +764 | <code>apply_controller_settings(true);</code> | Restore device and mode after game loading reset the core pad state. |
| +834 | <code>/* Apply a changed mode on resume without changing the active device. */</code> | Source comment explaining the adjacent change: Apply a changed mode on resume without changing the active device. |
| +835 | <code>apply_controller_settings(false);</code> | Apply mode edits on resume while keeping the active device until restart. |
| +854 | <code>/* Reloading resets pad state, so restore the selected device and mode. */</code> | Source comment explaining the adjacent change: Reloading resets pad state, so restore the selected device and mode. |
| +855 | <code>apply_controller_settings(true);</code> | Restore device and mode after game loading reset the core pad state. |

### [libpcsxcore/pad.c](../libpcsxcore/pad.c)

| Line | Changed code | Purpose |
| --- | --- | --- |
| +612 | <code>/* Set the requested mode deterministically instead of toggling an unknown game-selected mode. */</code> | Source comment explaining the adjacent change: Set the requested mode deterministically instead of toggling an unknown game-selected mode. |
| +613 | <code>int padSetAnalog(unsigned int index, int analog)</code> | Define or declare the explicit per-port analog-mode setter. |
| +614 | <code>{</code> | Open or close the surrounding block/declaration; no independent operation. |
| +615 | <code>if (index &gt;= sizeof(g.pads) / sizeof(g.pads[0]))</code> | Reject ports outside the fixed pad array before accessing memory. |
| +616 | <code>return -1;</code> | Report an invalid controller index to the caller. |
| +617 | <code>/* Normalize callers to the one-bit protocol mode and mark a user-initiated change. */</code> | Source comment explaining the adjacent change: Normalize callers to the one-bit protocol mode and mark a user-initiated change. |
| +618 | <code>g.pads[index].ds.padMode = !!analog;</code> | Normalize the requested mode to zero or one. |
| +619 | <code>g.pads[index].ds.userToggled = 1;</code> | Mark the mode change as user initiated, matching the existing toggle API. |
| +620 | <code>return g.pads[index].ds.padMode;</code> | Return the resulting controller mode. |
| +621 | <code>}</code> | Open or close the surrounding block/declaration; no independent operation. |
| +622 | <code>(blank)</code> | Visual separation only; no runtime effect. |

### [libpcsxcore/plugins.h](../libpcsxcore/plugins.h)

| Line | Changed code | Purpose |
| --- | --- | --- |
| +138 | <code>/* Explicit mode setter for the Vita menu; returns -1 for an invalid port. */</code> | Source comment explaining the adjacent change: Explicit mode setter for the Vita menu; returns -1 for an invalid port. |
| +139 | <code>int padSetAnalog(unsigned int index, int analog);</code> | Define or declare the explicit per-port analog-mode setter. |

## feat(vita): make gameplay Cross and Circle swapping optional

Commit: `75e80d3f`

Add a persisted, default-off menu option. Remap a snapshot of the two original bits on each gameplay poll, preserving simultaneous presses and every other button. Menu input remains independent.

### [frontend/vita/rom_selector.cpp](../frontend/vita/rom_selector.cpp)

| Line | Changed code | Purpose |
| --- | --- | --- |
| +203 | <code>{ &quot;SYSTEM&quot;, &quot;Swap Cross / Circle&quot;, &quot;Swap Cross (X) and Circle (O) for game input, including gameplay actions. Emulator menu controls are unchanged. Applies immediately on resume.&quot;, &quot;vita_swap_cross_circle&quot;, k_toggle_choices, 2, false },</code> | Add the SYSTEM toggle, its help text, persistence key, and two enabled/disabled choices. |

### [frontend/vita/settings.c](../frontend/vita/settings.c)

| Line | Changed code | Purpose |
| --- | --- | --- |
| +21 | <code>/* Leave face-button mapping unchanged until the player explicitly opts in. */</code> | Source comment explaining the adjacent change: Leave face-button mapping unchanged until the player explicitly opts in. |
| +22 | <code>{ &quot;vita_swap_cross_circle&quot;, &quot;disabled&quot;, {0} },</code> | Persist face-button swapping with disabled as the default. |

### [frontend/vita/standalone.c](../frontend/vita/standalone.c)

| Line | Changed code | Purpose |
| --- | --- | --- |
| +411 | <code>/* Remap only the gameplay snapshot; menu input uses its own controller reads. */</code> | Source comment explaining the adjacent change: Remap only the gameplay snapshot; menu input uses its own controller reads. |
| +412 | <code>const char *swap = VitaSettings_GetOption(&quot;vita_swap_cross_circle&quot;);</code> | Read the current face-button swap option on every gameplay poll. |
| +413 | <code>if (swap &amp;&amp; strcmp(swap, &quot;enabled&quot;) == 0) {</code> | Remap only when a valid enabled option is present. |
| +414 | <code>/* Snapshot both original bits before clearing them, including simultaneous presses. */</code> | Source comment explaining the adjacent change: Snapshot both original bits before clearing them, including simultaneous presses. |
| +415 | <code>const uint32_t buttons = g_pad.buttons;</code> | Snapshot the original bits before replacing either one. |
| +416 | <code>/* Preserve every other button while replacing the two face-button bits. */</code> | Source comment explaining the adjacent change: Preserve every other button while replacing the two face-button bits. |
| +417 | <code>g_pad.buttons &amp;= ~(SCE_CTRL_CROSS &#124; SCE_CTRL_CIRCLE);</code> | Clear Cross and Circle while preserving every other button. |
| +418 | <code>if (buttons &amp; SCE_CTRL_CROSS)</code> | Check whether Cross was originally pressed. |
| +419 | <code>g_pad.buttons &#124;= SCE_CTRL_CIRCLE;</code> | Emit Circle for an original Cross press. |
| +420 | <code>if (buttons &amp; SCE_CTRL_CIRCLE)</code> | Check whether Circle was originally pressed independently of Cross. |
| +421 | <code>g_pad.buttons &#124;= SCE_CTRL_CROSS;</code> | Emit Cross for an original Circle press, including simultaneous presses. |
| +422 | <code>}</code> | Open or close the surrounding block/declaration; no independent operation. |

## feat(vita): map lower touchscreen corners to L2 and R2

Commit: `5e00683a`

Discover front-panel bounds and enable sampling only after validation. Combine contacts in the outer thirds of the bottom third into trigger bits. Recompute every poll so release is immediate while physical trigger bits remain intact.

### [frontend/vita/standalone.c](../frontend/vita/standalone.c)

| Line | Changed code | Purpose |
| --- | --- | --- |
| +11 | <code>#include &lt;psp2/touch.h&gt;</code> | Declare the Vita touchscreen types, constants, and SDK functions. |
| +65 | <code>/* Cache the panel bounds so touch coordinates do not assume a fixed resolution. */</code> | Source comment explaining the adjacent change: Cache the panel bounds so touch coordinates do not assume a fixed resolution. |
| +66 | <code>static SceTouchPanelInfo g_front_touch_panel;</code> | Retain the discovered touchscreen bounds for coordinate normalization. |
| +67 | <code>/* Ignore touchscreen input unless panel discovery and sampling both succeeded. */</code> | Source comment explaining the adjacent change: Ignore touchscreen input unless panel discovery and sampling both succeeded. |
| +68 | <code>static bool g_front_touch_ready;</code> | Track whether panel validation and sampling setup succeeded. |
| +410 | <code>static uint32_t front_touch_buttons(void)</code> | Translate a fresh touchscreen report into emulated trigger bits. |
| +411 | <code>{</code> | Open or close the surrounding block/declaration; no independent operation. |
| +412 | <code>SceTouchData touch;</code> | Allocate the SDK touch-report buffer for this poll. |
| +413 | <code>uint32_t buttons = 0;</code> | Start with no synthetic buttons so released contacts do not remain pressed. |
| +414 | <code>if (!g_front_touch_ready &#124;&#124;</code> | Skip touch input if initialization failed. |
| +415 | <code>sceTouchPeek(SCE_TOUCH_PORT_FRONT, &amp;touch, 1) &lt;= 0)</code> | Read the latest front-panel report without waiting; no report produces no synthetic buttons. |
| +416 | <code>return 0;</code> | Return no synthetic trigger presses when touch data is unavailable. |
| +417 | <code>(blank)</code> | Visual separation only; no runtime effect. |
| +418 | <code>/* Panel maxima are inclusive; normalize all contacts to this rectangle. */</code> | Source comment explaining the adjacent change: Panel maxima are inclusive; normalize all contacts to this rectangle. |
| +419 | <code>const int width = g_front_touch_panel.maxDispX - g_front_touch_panel.minDispX + 1;</code> | Compute panel width from inclusive horizontal bounds. |
| +420 | <code>const int height = g_front_touch_panel.maxDispY - g_front_touch_panel.minDispY + 1;</code> | Compute panel height from inclusive vertical bounds. |
| +421 | <code>/* Combine simultaneous contacts while respecting the fixed report capacity. */</code> | Source comment explaining the adjacent change: Combine simultaneous contacts while respecting the fixed report capacity. |
| +422 | <code>for (unsigned i = 0; i &lt; touch.reportNum &amp;&amp; i &lt; SCE_TOUCH_MAX_REPORT; ++i) {</code> | Process all reported contacts up to the SDK array capacity. |
| +423 | <code>const int x = touch.report[i].x - g_front_touch_panel.minDispX;</code> | Normalize the contact x coordinate to the panel origin. |
| +424 | <code>const int y = touch.report[i].y - g_front_touch_panel.minDispY;</code> | Normalize the contact y coordinate to the panel origin. |
| +425 | <code>/* Outer thirds of the bottom third; center and upper area are inert. */</code> | Source comment explaining the adjacent change: Outer thirds of the bottom third; center and upper area are inert. |
| +426 | <code>if (x &lt; 0 &#124;&#124; x &gt;= width &#124;&#124; y &lt; (height * 2) / 3 &#124;&#124; y &gt;= height)</code> | Reject off-panel contacts and contacts above the bottom third. |
| +427 | <code>continue;</code> | Ignore this contact and examine the next one. |
| +428 | <code>if (x &lt; width / 3)</code> | Recognize the left third of the accepted bottom region. |
| +429 | <code>buttons &#124;= SCE_CTRL_L2;</code> | Add L2 without removing any trigger bit from another contact. |
| +430 | <code>else if (x &gt;= (width * 2) / 3)</code> | Recognize the right third, leaving the center inactive. |
| +431 | <code>buttons &#124;= SCE_CTRL_R2;</code> | Add R2, allowing simultaneous left/right contacts. |
| +432 | <code>}</code> | Open or close the surrounding block/declaration; no independent operation. |
| +433 | <code>return buttons;</code> | Return the union of synthetic trigger presses for this poll. |
| +434 | <code>}</code> | Open or close the surrounding block/declaration; no independent operation. |
| +435 | <code>(blank)</code> | Visual separation only; no runtime effect. |
| +442 | <code>/* Recomputed each poll: releasing a touch releases its button, while</code> | Source comment explaining the adjacent change: Recomputed each poll: releasing a touch releases its button, while |
| +443 | <code>* physical/external-controller L2 and R2 remain available. */</code> | Source comment explaining the adjacent change: physical/external-controller L2 and R2 remain available. |
| +444 | <code>g_pad.buttons &#124;= front_touch_buttons();</code> | Merge synthetic triggers with the physical controller snapshot. |
| +445 | <code>(blank)</code> | Visual separation only; no runtime effect. |
| +727 | <code>/* Short-circuit failures: never consume touch reports from an invalid panel. */</code> | Source comment explaining the adjacent change: Short-circuit failures: never consume touch reports from an invalid panel. |
| +728 | <code>g_front_touch_ready =</code> | Store the combined success of panel discovery, validation, and sampling setup. |
| +729 | <code>sceTouchGetPanelInfo(SCE_TOUCH_PORT_FRONT, &amp;g_front_touch_panel) &gt;= 0 &amp;&amp;</code> | Read front-panel bounds and require SDK success. |
| +730 | <code>g_front_touch_panel.maxDispX &gt; g_front_touch_panel.minDispX &amp;&amp;</code> | Require a positive horizontal coordinate extent. |
| +731 | <code>g_front_touch_panel.maxDispY &gt; g_front_touch_panel.minDispY &amp;&amp;</code> | Require a positive vertical coordinate extent. |
| +732 | <code>sceTouchSetSamplingState(SCE_TOUCH_PORT_FRONT, SCE_TOUCH_SAMPLING_STATE_START) &gt;= 0;</code> | Start front-touch sampling only after the preceding checks pass. |

## diagnostics(vita): record input and renderer startup state

Commit: `0c9656e2`

Persist frontend and renderer messages, sample analog input and presentation state, and record skipped/submitted frames. Keep the original diagnostic behavior, including glGetError consumption and the presentation counter semantics, separate from functional fixes.

### [frontend/vita/standalone.c](../frontend/vita/standalone.c)

| Line | Changed code | Purpose |
| --- | --- | --- |
| +103 | <code>/* Keep one append stream for frontend diagnostics during this process. */</code> | Source comment explaining the adjacent change: Keep one append stream for frontend diagnostics during this process. |
| +104 | <code>static FILE *g_debug_log;</code> | Keep a reusable frontend diagnostic file stream. |
| +109 | <code>(blank)</code> | Visual separation only; no runtime effect. |
| +110 | <code>static void debug_log(const char *fmt, ...)</code> | Provide a variadic helper for persistent frontend diagnostics. |
| +111 | <code>{</code> | Open or close the surrounding block/declaration; no independent operation. |
| +112 | <code>va_list ap;</code> | Declare the variable-argument cursor used to format the message. |
| +113 | <code>if (!g_debug_log)</code> | Open the log lazily, or skip writing if opening failed. |
| +114 | <code>g_debug_log = fopen(PCSX_AWO_APP_ROOT &quot;/renderer.log&quot;, &quot;ab&quot;);</code> | Append frontend diagnostics under the application root without truncating older output. |
| +115 | <code>if (!g_debug_log)</code> | Open the log lazily, or skip writing if opening failed. |
| +116 | <code>return;</code> | Skip this diagnostic write when no log stream is available. |
| +117 | <code>va_start(ap, fmt);</code> | Initialize a fresh argument cursor for this output pass. |
| +118 | <code>vfprintf(g_debug_log, fmt, ap);</code> | Format the frontend message into the persistent log. |
| +119 | <code>va_end(ap);</code> | Release the argument cursor after formatting. |
| +120 | <code>fflush(g_debug_log);</code> | Flush buffered diagnostics so recent messages survive an abnormal exit more reliably. |
| +121 | <code>}</code> | Open or close the surrounding block/declaration; no independent operation. |
| +453 | <code>static unsigned analog_log_frames;</code> | Keep a process-lifetime count of sampled DualShock polls. |
| +461 | <code>(blank)</code> | Visual separation only; no runtime effect. |
| +462 | <code>/* Sample once per 60 DualShock polls and stop after 3600 such polls. */</code> | Source comment explaining the adjacent change: Sample once per 60 DualShock polls and stop after 3600 such polls. |
| +463 | <code>if (g_controller_dualshock &amp;&amp; analog_log_frames &lt; 3600) {</code> | Limit input tracing to the first 3600 polls with DualShock active. |
| +464 | <code>if ((analog_log_frames++ % 60) == 0)</code> | Log every 60th eligible poll while advancing the counter each eligible poll. |
| +465 | <code>debug_log(&quot;PCSX AwO: analog input left=%u,%u right=%u,%u requested=%s\n&quot;,</code> | Write the diagnostic described by this format string; continued lines supply its values. |
| +466 | <code>g_pad.lx, g_pad.ly, g_pad.rx, g_pad.ry,</code> | Supply both raw stick coordinate pairs to the input diagnostic. |
| +467 | <code>g_controller_analog ? &quot;analog&quot; : &quot;digital&quot;);</code> | Format the cached user-requested mode, which may differ from a later game-selected mode. |
| +468 | <code>}</code> | Open or close the surrounding block/declaration; no independent operation. |
| +746 | <code>debug_log(&quot;\nPCSX AwO: main entered [diag-v3, packed vertex ABI fixed, PCSX async GPU disabled]\n&quot;);</code> | Write the diagnostic described by this format string; continued lines supply its values. |
| +748 | <code>debug_log(&quot;PCSX AwO: renderer setting=%s\n&quot;,</code> | Write the diagnostic described by this format string; continued lines supply its values. |
| +749 | <code>VitaSettings_GetOption(&quot;vita_renderer&quot;));</code> | Supply the selected renderer value to the startup message. |
| -725 | <code>/* Ext2 polling needs extended analog sampling to supply both stick axes. */</code> | Source comment explaining the adjacent change: Ext2 polling needs extended analog sampling to supply both stick axes. |
| -726 | <code>sceCtrlSetSamplingModeExt(SCE_CTRL_MODE_ANALOG_WIDE);</code> | Enable wide analog sampling for the extended controller API; diagnostics also record the return value. |
| +752 | <code>/* Log the SDK result while enabling the extended analog sampling path. */</code> | Source comment explaining the adjacent change: Log the SDK result while enabling the extended analog sampling path. |
| +753 | <code>debug_log(&quot;PCSX AwO: extended analog sampling result=%d\n&quot;,</code> | Write the diagnostic described by this format string; continued lines supply its values. |
| +754 | <code>sceCtrlSetSamplingModeExt(SCE_CTRL_MODE_ANALOG_WIDE));</code> | Enable wide analog sampling for the extended controller API; diagnostics also record the return value. |
| +761 | <code>debug_log(&quot;PCSX AwO: front touch L2/R2 %s\n&quot;,</code> | Write the diagnostic described by this format string; continued lines supply its values. |
| +762 | <code>g_front_touch_ready ? &quot;enabled&quot; : &quot;unavailable&quot;);</code> | Format whether front-touch trigger initialization succeeded. |
| +778 | <code>debug_log(&quot;PCSX AwO: video initialized\n&quot;);</code> | Write the diagnostic described by this format string; continued lines supply its values. |
| +795 | <code>debug_log(&quot;PCSX AwO: retro_init complete\n&quot;);</code> | Write the diagnostic described by this format string; continued lines supply its values. |
| +840 | <code>debug_log(&quot;PCSX AwO: booting %s\n&quot;, game_path);</code> | Write the diagnostic described by this format string; continued lines supply its values. |
| +845 | <code>debug_log(&quot;PCSX AwO: retro_load_game failed\n&quot;);</code> | Write the diagnostic described by this format string; continued lines supply its values. |
| +849 | <code>debug_log(&quot;PCSX AwO: retro_load_game complete\n&quot;);</code> | Write the diagnostic described by this format string; continued lines supply its values. |
| +853 | <code>debug_log(&quot;PCSX AwO: port 1 controller=%s\n&quot;, g_controller_dualshock ? &quot;dualshock&quot; : &quot;standard&quot;);</code> | Write the diagnostic described by this format string; continued lines supply its values. |
| +975 | <code>debug_log(&quot;PCSX AwO: shutting down\n&quot;);</code> | Write the diagnostic described by this format string; continued lines supply its values. |
| +976 | <code>if (g_debug_log)</code> | Close the frontend stream only if it was successfully opened. |
| +977 | <code>fclose(g_debug_log);</code> | Flush and release the frontend diagnostic file at shutdown. |
| +978 | <code>g_debug_log = NULL;</code> | Clear the closed frontend stream pointer. |

### [plugins/gpu_vita_ctr/gpulib_if.c](../plugins/gpu_vita_ctr/gpulib_if.c)

| Line | Changed code | Purpose |
| --- | --- | --- |
| +42 | <code>/* Mirror renderer messages to an append-only file for on-device diagnosis. */</code> | Source comment explaining the adjacent change: Mirror renderer messages to an append-only file for on-device diagnosis. |
| +43 | <code>static FILE *s_renderer_log;</code> | Keep a reusable renderer diagnostic file stream. |
| +55 | <code>static unsigned s_present_debug_count;</code> | Count frames reaching the submission path for presentation diagnostics. |
| -88 | <code>va_list ap;</code> | Declare the variable-argument cursor used to format the message. |
| -89 | <code>va_start(ap, fmt);</code> | Initialize a fresh argument cursor for this output pass. |
| -90 | <code>vfprintf(stdout, fmt, ap);</code> | Preserve the normal console log output. |
| -91 | <code>va_end(ap);</code> | Release the argument cursor after formatting. |
| +91 | <code>va_list ap;</code> | Declare the variable-argument cursor used to format the message. |
| +92 | <code>va_start(ap, fmt);</code> | Initialize a fresh argument cursor for this output pass. |
| +93 | <code>vfprintf(stdout, fmt, ap);</code> | Preserve the normal console log output. |
| +94 | <code>va_end(ap);</code> | Release the argument cursor after formatting. |
| +95 | <code>if (!s_renderer_log)</code> | Open the renderer log lazily when no stream exists yet. |
| +96 | <code>s_renderer_log = fopen(PCSX_AWO_APP_ROOT &quot;/renderer.log&quot;, &quot;ab&quot;);</code> | Append renderer diagnostics under the application root without truncating older output. |
| +97 | <code>if (s_renderer_log)</code> | Write the file copy only when opening the stream succeeded. |
| +98 | <code>{</code> | Open or close the surrounding block/declaration; no independent operation. |
| +99 | <code>va_start(ap, fmt);</code> | Initialize a fresh argument cursor for this output pass. |
| +100 | <code>vfprintf(s_renderer_log, fmt, ap);</code> | Format a second copy of the renderer message into the file. |
| +101 | <code>va_end(ap);</code> | Release the argument cursor after formatting. |
| +102 | <code>fflush(s_renderer_log);</code> | Flush buffered diagnostics so recent messages survive an abnormal exit more reliably. |
| +103 | <code>}</code> | Open or close the surrounding block/declaration; no independent operation. |
| -96 | <code>va_list ap;</code> | Declare the variable-argument cursor used to format the message. |
| -97 | <code>va_start(ap, fmt);</code> | Initialize a fresh argument cursor for this output pass. |
| -98 | <code>vfprintf(stderr, fmt, ap);</code> | Preserve error/warning output to the error stream. |
| -99 | <code>va_end(ap);</code> | Release the argument cursor after formatting. |
| +108 | <code>va_list ap;</code> | Declare the variable-argument cursor used to format the message. |
| +109 | <code>va_start(ap, fmt);</code> | Initialize a fresh argument cursor for this output pass. |
| +110 | <code>vfprintf(stderr, fmt, ap);</code> | Preserve error/warning output to the error stream. |
| +111 | <code>va_end(ap);</code> | Release the argument cursor after formatting. |
| +112 | <code>if (!s_renderer_log)</code> | Open the renderer log lazily when no stream exists yet. |
| +113 | <code>s_renderer_log = fopen(PCSX_AWO_APP_ROOT &quot;/renderer.log&quot;, &quot;ab&quot;);</code> | Append renderer diagnostics under the application root without truncating older output. |
| +114 | <code>if (s_renderer_log)</code> | Write the file copy only when opening the stream succeeded. |
| +115 | <code>{</code> | Open or close the surrounding block/declaration; no independent operation. |
| +116 | <code>va_start(ap, fmt);</code> | Initialize a fresh argument cursor for this output pass. |
| +117 | <code>vfprintf(s_renderer_log, fmt, ap);</code> | Format a second copy of the renderer message into the file. |
| +118 | <code>va_end(ap);</code> | Release the argument cursor after formatting. |
| +119 | <code>fflush(s_renderer_log);</code> | Flush buffered diagnostics so recent messages survive an abnormal exit more reliably. |
| +120 | <code>}</code> | Open or close the surrounding block/declaration; no independent operation. |
| -104 | <code>va_list ap;</code> | Declare the variable-argument cursor used to format the message. |
| -105 | <code>va_start(ap, fmt);</code> | Initialize a fresh argument cursor for this output pass. |
| -106 | <code>vfprintf(stderr, fmt, ap);</code> | Preserve error/warning output to the error stream. |
| -107 | <code>va_end(ap);</code> | Release the argument cursor after formatting. |
| +125 | <code>va_list ap;</code> | Declare the variable-argument cursor used to format the message. |
| +126 | <code>va_start(ap, fmt);</code> | Initialize a fresh argument cursor for this output pass. |
| +127 | <code>vfprintf(stderr, fmt, ap);</code> | Preserve error/warning output to the error stream. |
| +128 | <code>va_end(ap);</code> | Release the argument cursor after formatting. |
| +129 | <code>if (!s_renderer_log)</code> | Open the renderer log lazily when no stream exists yet. |
| +130 | <code>s_renderer_log = fopen(PCSX_AWO_APP_ROOT &quot;/renderer.log&quot;, &quot;ab&quot;);</code> | Append renderer diagnostics under the application root without truncating older output. |
| +131 | <code>if (s_renderer_log)</code> | Write the file copy only when opening the stream succeeded. |
| +132 | <code>{</code> | Open or close the surrounding block/declaration; no independent operation. |
| +133 | <code>va_start(ap, fmt);</code> | Initialize a fresh argument cursor for this output pass. |
| +134 | <code>vfprintf(s_renderer_log, fmt, ap);</code> | Format a second copy of the renderer message into the file. |
| +135 | <code>va_end(ap);</code> | Release the argument cursor after formatting. |
| +136 | <code>fflush(s_renderer_log);</code> | Flush buffered diagnostics so recent messages survive an abnormal exit more reliably. |
| +137 | <code>}</code> | Open or close the surrounding block/declaration; no independent operation. |
| +185 | <code>/* Trace initial frames, then sample periodically through frame 3600. */</code> | Source comment explaining the adjacent change: Trace initial frames, then sample periodically through frame 3600. |
| +186 | <code>static unsigned diagnostic_frame;</code> | Keep an end-scene invocation counter across frames. |
| +187 | <code>const unsigned frame = ++diagnostic_frame;</code> | Advance and snapshot the end-scene counter for this invocation. |
| +188 | <code>const int trace = frame &lt;= 12 &#124;&#124; (frame &lt;= 3600 &amp;&amp; frame % 120 == 0);</code> | Trace the first 12 scenes and every 120th scene through scene 3600. |
| +193 | <code>if (trace)</code> | Emit the following scene diagnostic only for the sampled scenes. |
| +194 | <code>Platform_Log(&quot;[diag-v2] end-scene %u blank=%d rgb24=%d scanout=%d display=%d,%d %dx%d clip=%d,%d %dx%d dfe=%d\n&quot;,</code> | Write the diagnostic described by this format string; continued lines supply its values. |
| +195 | <code>frame, blank, display-&gt;isrgb24, use_vram_scanout,</code> | Record the scene number, blanking, pixel format, and scanout choice. |
| +196 | <code>display-&gt;disp.x, display-&gt;disp.y, display-&gt;disp.w, display-&gt;disp.h,</code> | Record the emulated display rectangle. |
| +197 | <code>draw-&gt;clip.x, draw-&gt;clip.y, draw-&gt;clip.w, draw-&gt;clip.h, draw-&gt;dfe);</code> | Record the draw clip and draw-to-display enable flag. |
| +198 | <code>(blank)</code> | Visual separation only; no runtime effect. |
| +220 | <code>if (trace)</code> | Emit the following scene diagnostic only for the sampled scenes. |
| +221 | <code>Platform_Log(&quot;[diag-v2] end-scene %u swap returned gl_error=0x%x\n&quot;, frame, (unsigned)glGetError());</code> | Record the swap result and read/clear the current GL error flag. |
| -758 | <code>if (!s_context_ready)</code> | Preserve the no-context early return; the new block adds a diagnostic before returning. |
| +800 | <code>/* Skipped frames do not advance the submission-attempt counter. */</code> | Source comment explaining the adjacent change: Skipped frames do not advance the submission-attempt counter. |
| +801 | <code>if (!s_context_ready) {</code> | Preserve the no-context early return; the new block adds a diagnostic before returning. |
| +802 | <code>if (s_present_debug_count &lt; 8)</code> | Emit initial presentation diagnostics; skipped attempts do not increment this counter. |
| +803 | <code>Platform_Log(&quot;PCSX AwO: High Octane present skipped: context not ready\n&quot;);</code> | Write the diagnostic described by this format string; continued lines supply its values. |
| +805 | <code>}</code> | Open or close the surrounding block/declaration; no independent operation. |
| -764 | <code>if (display_width &lt;= 0 &#124;&#124; display_height &lt;= 0 &#124;&#124; w &lt;= 0 &#124;&#124; h &lt;= 0)</code> | Preserve rejection of invalid dimensions while adding a diagnostic. |
| +810 | <code>if (display_width &lt;= 0 &#124;&#124; display_height &lt;= 0 &#124;&#124; w &lt;= 0 &#124;&#124; h &lt;= 0) {</code> | Preserve rejection of invalid dimensions while adding a diagnostic. |
| +811 | <code>if (s_present_debug_count &lt; 8)</code> | Emit initial presentation diagnostics; skipped attempts do not increment this counter. |
| +812 | <code>Platform_Log(&quot;PCSX AwO: High Octane present skipped: invalid dimensions %dx%d -&gt; %dx%d\n&quot;,</code> | Write the diagnostic described by this format string; continued lines supply its values. |
| +813 | <code>display_width, display_height, w, h);</code> | Provide display and source dimensions for the skipped-frame message. |
| +815 | <code>}</code> | Open or close the surrounding block/declaration; no independent operation. |
| -809 | <code>NativeGpu_SubmitFrontendFrame();</code> | The previous code submitted the frame without recording the returned status. |
| +859 | <code>{</code> | Open or close the surrounding block/declaration; no independent operation. |
| +860 | <code>const int submitted = NativeGpu_SubmitFrontendFrame();</code> | Submit the frame as before and retain the returned status for logging. |
| +861 | <code>if (s_present_debug_count &lt; 8) {</code> | Emit initial presentation diagnostics; skipped attempts do not increment this counter. |
| +862 | <code>Platform_Log(&quot;PCSX AwO: High Octane present #%u: src=%d,%d display=%dx%d rect=%d,%d %dx%d submitted=%d\n&quot;,</code> | Write the diagnostic described by this format string; continued lines supply its values. |
| +863 | <code>s_present_debug_count + 1, src_x, src_y, display_width, display_height,</code> | Record the submission-attempt number, source origin, and display size. |
| +864 | <code>x, y, w, h, submitted);</code> | Record the destination rectangle and submission status. |
| +865 | <code>}</code> | Open or close the surrounding block/declaration; no independent operation. |
| +866 | <code>s_present_debug_count++;</code> | Advance after reaching submission even if the submitted result is zero. |
| +867 | <code>}</code> | Open or close the surrounding block/declaration; no independent operation. |

### [plugins/gpu_vita_ctr/native_gpu.c](../plugins/gpu_vita_ctr/native_gpu.c)

| Line | Changed code | Purpose |
| --- | --- | --- |
| +2681 | <code>/* Sample geometry counts and synchronous mode without logging every batch. */</code> | Source comment explaining the adjacent change: Sample geometry counts and synchronous mode without logging every batch. |
| +2682 | <code>static unsigned diagnostic_batch;</code> | Keep a process-lifetime prepared-geometry batch counter. |
| +2683 | <code>const unsigned batch = ++diagnostic_batch;</code> | Advance and snapshot the current batch number. |
| +2684 | <code>if (batch &lt;= 12 &#124;&#124; (batch &lt;= 7200 &amp;&amp; batch % 240 == 0)) {</code> | Trace the first 12 batches and every 240th batch through batch 7200. |
| +2685 | <code>Platform_Log(&quot;[diag-v2] draw batch=%u vertices=%d splits=%d synchronous=%d\n&quot;,</code> | Write the diagnostic described by this format string; continued lines supply its values. |
| +2686 | <code>batch, vertexCount, splitCount, NativeGpu_IsSynchronousFrame());</code> | Record geometry counts and whether this batch uses synchronous rendering. |
| +2687 | <code>}</code> | Open or close the surrounding block/declaration; no independent operation. |

### [plugins/gpu_vita_ctr/native_renderer.c](../plugins/gpu_vita_ctr/native_renderer.c)

| Line | Changed code | Purpose |
| --- | --- | --- |
| +4027 | <code>/* Capture target/viewport state; glGetError also consumes the current error flag. */</code> | Source comment explaining the adjacent change: Capture target/viewport state; glGetError also consumes the current error flag. |
| +4028 | <code>static unsigned diagnostic_present;</code> | Keep a process-lifetime main-target presentation counter. |
| +4029 | <code>const unsigned present = ++diagnostic_present;</code> | Advance and snapshot the main-target presentation number. |
| +4030 | <code>if (present &lt;= 12 &#124;&#124; (present &lt;= 3600 &amp;&amp; present % 120 == 0)) {</code> | Trace the first 12 presents and every 120th present through present 3600. |
| +4031 | <code>Platform_Log(&quot;[diag-v2] main present=%u texture=%u fbo=%u physical=%dx%d logical=%dx%d viewport=%d,%d %dx%d shader=%u gl_error=0x%x\n&quot;,</code> | Write the diagnostic described by this format string; continued lines supply its values. |
| +4032 | <code>present, (unsigned)s_mainRenderTarget.texture, (unsigned)s_mainRenderTarget.framebuffer,</code> | Record the present number and active texture/framebuffer handles. |
| +4033 | <code>s_mainRenderTarget.width, s_mainRenderTarget.height,</code> | Record the physical render-target dimensions. |
| +4034 | <code>s_mainRenderTarget.logicalWidth, s_mainRenderTarget.logicalHeight,</code> | Record the emulated logical render-target dimensions. |
| +4035 | <code>s_presentViewport.x, s_presentViewport.y, s_presentViewport.w, s_presentViewport.h,</code> | Record the destination viewport origin and size. |
| +4036 | <code>(unsigned)s_presentRgbaShader, (unsigned)glGetError());</code> | Record the shader and consume the current GL error flag for the diagnostic. |
| +4037 | <code>}</code> | Open or close the surrounding block/declaration; no independent operation. |

## chore(vita): ignore generated packages and local build artifacts

Commit: `8f4cbf67`

Exclude ELF/VELF/VPK products, local build logs, and the Makefile backup from accidental staging without deleting those files.

### [.gitignore](../.gitignore)

| Line | Changed code | Purpose |
| --- | --- | --- |
| +49 | <code>(blank)</code> | Visual separation only; no runtime effect. |
| +50 | <code># PCSX AwO build outputs and local development artifacts</code> | Source comment explaining the adjacent change: PCSX AwO build outputs and local development artifacts |
| +51 | <code>/pcsx-awo.elf</code> | Exclude this root-level generated artifact/pattern from ordinary Git staging; existing files are not deleted. |
| +52 | <code>/pcsx-awo.velf</code> | Exclude this root-level generated artifact/pattern from ordinary Git staging; existing files are not deleted. |
| +53 | <code>/pcsx-awo.vpk</code> | Exclude this root-level generated artifact/pattern from ordinary Git staging; existing files are not deleted. |
| +54 | <code>/*-build.log</code> | Source comment explaining the adjacent change: -build.log |
| +55 | <code>/build.log</code> | Exclude this root-level generated artifact/pattern from ordinary Git staging; existing files are not deleted. |
| +56 | <code>/Makefile.libretro.bak</code> | Exclude this root-level generated artifact/pattern from ordinary Git staging; existing files are not deleted. |


## Validation and remaining checks

- The Vita core and frontend were compiled in the fresh clone with `make -f Makefile.vita -j4`; ELF, VELF, and VPK packaging passed. A subsequent rebuild after adding braces to diagnostic blocks also passed.
- Existing vitaGL/SDK libraries were reused. The 26 compiled post-processing shaders were reused only after comparing their corresponding shader sources byte-for-byte with this clone. Shader compilation itself was not revalidated.
- `python3 tests/vita_gpu_visible_subviewport_test.py` passed.
- The LiveArea test was compiled with `cc -std=c99 -Wall -Wextra -Werror tests/vita_livearea_launch_test.c frontend/vita/livearea_launch.c -o build-vita/vita_livearea_launch_test`; running it passed.
- Diff whitespace checks passed with `cr-at-eol` enabled to respect the existing CRLF header format.
- The build reports executable-stack warnings, a jobserver warning, an unused dynarec helper, and a ROM-selector format-truncation warning. The two diagnostic indentation warnings found during the first build were resolved in the diagnostics commit.
- These checks cover the final series, not a full build of each intermediate commit. Vita hardware checks remain: both analog sticks, mode changes and game overrides, Cross/Circle swapping including simultaneous presses, touchscreen L2/R2 press/release and simultaneous contacts, external triggers, menus, both renderers, and LiveArea launching.
