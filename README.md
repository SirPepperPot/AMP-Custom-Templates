# AMP Custom Templates

Custom CubeCoders AMP templates, including a dedicated YgoMaster server for Linux.

## YgoMaster (template version 3)

The YgoMaster template launches `wine MonoRun.exe YgoMaster.exe` from the
`ygomaster/YgoMaster` directory. This is the setup verified to load the
Windows `duel.dll` and complete multiplayer duels on Ubuntu 24.04 with Wine 11.

**This template does not install Wine, Xvfb, Master Duel, or YgoMaster files.**
It does **not** automatically start the Xvfb display. The display must be kept
running independently (the included systemd unit below is recommended).

### Expected installation tree

```text
AMP instance/
  ygomaster/
    masterduel.exe
    masterduel_Data/Plugins/x86_64/duel.dll
    YgoMaster/
      MonoRun.exe
      mono/
      YgoMaster.exe
      YgoMasterClient.exe
      Data/
        Settings.json
        CardData/
        Players/
```

Get `MonoRun.exe` and `mono/` from the
[official YgoMaster Linux download](https://github.com/pixeltris/YgoMaster/releases/download/v1.50/YgoMaster-Linux-Data-v1.zip).
Use Master Duel files from a legitimate installation.
[Official Linux instructions](https://github.com/pixeltris/YgoMaster/blob/master/Docs/Linux.md).

### Prerequisites on the AMP Linux host

- A working 64-bit Wine installation (tested with Wine 11).
- `xvfb` and `xauth` (`sudo apt install xvfb xauth`).
- The `amp` user must be able to read and write YgoMaster's `Data/Players` directory.
- The working Wine prefix must be maintained at `/home/amp/.wine-ygomaster-test`.
  This name was retained deliberately to reuse the already-tested prefix and avoid
  resetting the server environment. Don't delete it merely because it says 'test'.
- The private runtime directory must exist: `sudo install -d -o amp -g amp -m 700 /home/amp/.runtime-ygomaster`.

This repository's template targets the **existing installation under /home/amp**.
For a different AMP service account, customize the environment variables in
`ygomaster.kvp` to point at the corresponding user-owned directories.

### Make the virtual display persistent

Only after stopping the manually launched Wine test and the old manually
launched Xvfb `:99` process, install the included service:

```bash
sudo install -m 644 ygomaster-xvfb.service /etc/systemd/system/ygomaster-xvfb.service
sudo systemctl daemon-reload
sudo systemctl enable --now ygomaster-xvfb.service
sudo systemctl status ygomaster-xvfb.service --no-pager
```

If it reports that display `:99` is already in use, check for an old manual
`Xvfb :99` process before restarting the service. Do not kill other Wine
processes belonging to unrelated AMP instances.

### Applying version 3 to an existing AMP instance

**Do not recreate the instance or overwrite `YgoMaster/Data/Players`.**
Updating the GitHub template alone doesn't change the instance currently
running in AMP. Before changing an existing instance, back up
`GenericModule.kvp`, `Data/Players`, and `Data/Settings.json`.

The active instance should launch with these settings:

```ini
App.RootDir=./ygomaster/
App.BaseDirectory=./ygomaster/
App.WorkingDir=YgoMaster
App.ExecutableLinux=/usr/bin/wine
App.LinuxCommandLineArgs=
App.CommandLineArgs=MonoRun.exe YgoMaster.exe
App.EnvironmentVariables={"DISPLAY":":99","XDG_RUNTIME_DIR":"/home/amp/.runtime-ygomaster","WINEPREFIX":"/home/amp/.wine-ygomaster-test","WINEARCH":"win64","WINEDEBUG":"-all"}
```

The previous AMP instance was observed reverting its startup values to
`GenericApplication/application` after an update. Therefore, **verify**
the active `GenericModule.kvp` after applying the revised template and before
attempting to start the game. This repository edit does not itself repair or
update an active instance.

A successful startup prints `(MonoRun)`, loads players, then prints
`Initialized`. Confirm the entire room duel works and that decks persist
after stopping/restarting the game server.

### Troubleshooting

- `Cannot open assembly YgoMaster.exe`: incorrect working directory or wrong
  launch method; the Wine template must launch `MonoRun.exe` from `YgoMaster`.
- `Data/CardData/#/CARD_IntID.bytes` missing: extract/copy the card data; the
  full game installation alone does not create these extracted files.
- `UnauthorizedAccessException Data/Players/.../Decks`: verify ownership
  and directory traverse (`x`) permissions as the `amp` user.
- PvP setup timeout and `nodrv_CreateWindow`: confirm the Xvfb `:99` display
  is active and that Wine processes inherit `DISPLAY=:99`.
- The Linux-native Mono launcher cannot load the native Windows `duel.dll`.
  Do not replace `/usr/bin/wine` with `/usr/bin/mono` for multiplayer.

The earlier `/usr/local/bin/YgoMaster.exe` shim (if installed) is not
required by the Wine + MonoRun method.
