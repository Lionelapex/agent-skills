# Windows Device and Driver Triage

## Short description

An agent that works out why a device on your PC is not working, finds the right official driver, checks that it is safe, and proves the fix worked before telling you it is done.

## Full description

Your Wi-Fi adapter stopped showing up. Your printer vanished. Device Manager has a yellow warning triangle next to something you do not recognise. Or you bought a cheap USB dongle and the only driver was on a disc you cannot use.

Most people at that point start downloading random driver packs from sites that bundle adware, and half the time the driver was never the problem.

This skill gives an agent a proper procedure instead.

### What the agent actually does

1. **Looks at the real hardware.** It queries Windows directly for what is plugged in and what is failing, rather than trusting a screenshot or a guess.
2. **Works out the type of fault.** A device that Windows cannot identify at all is a different problem from a device that is recognised but has no driver. These need completely different fixes, and confusing them wastes hours.
3. **Identifies unknown hardware.** If the device has no brand or model number, it uses the barcode on the box or the internal hardware ID to work out what chip is inside, and therefore which driver is correct.
4. **Finds the official driver.** It prefers Microsoft and the actual chip manufacturer over third-party driver sites.
5. **Checks the driver is safe.** It verifies the digital signature and the published checksum before running anything, and refuses anything it cannot account for.
6. **Explains the plan and waits.** You get a plain summary of what is wrong, what it will change, and how to undo it. Nothing is installed until you say yes.
7. **Proves it worked.** It checks the device is healthy, tests that it actually passes data, and checks again after a restart. A green tick in an installer is not treated as success.

### Why this is different

Most "driver helper" tools install first and ask questions later. This one is built around a single rule: classify the fault before touching a driver.

It is written to demand evidence. It will tell you plainly when a problem cannot be fixed by a driver at all, for example when a USB device is failing to be detected because of a bad port or a loose plug. That honesty saves people from reinstalling Windows over a five second fix.

### What you get

- `SKILL.md`, a complete procedure an agent can follow end to end.
- `description.md`, this listing text.
- Works with any agent that can read a Markdown file. No plugins, no accounts, no subscriptions.
- Uses only tools already built into Windows. Nothing extra to install in order to diagnose.

### Who it is for

- Home users who do not want to guess which driver is right.
- Anyone who bought hardware without a driver disc.
- IT support staff who want a repeatable checklist.
- Developers who want their coding agent to handle machine setup properly.

### Good to know

- Windows only.
- Installing a driver needs an administrator click. The skill tells you when that prompt is coming and why.
- It will never ask you for a password. You type those yourself.
- Some faults are physical and cannot be fixed in software. The skill says so instead of pretending otherwise.

## Category

System administration, Windows, troubleshooting

## Tags

windows, drivers, device manager, usb, wifi, hardware, troubleshooting, driver update, yellow triangle, unknown device, windows 11, it support

## Suggested pricing

Free. The full skill is published openly on GitHub, so a paid listing would compete with itself. Use this listing to reach people searching for the problem, and point them to the repository.
