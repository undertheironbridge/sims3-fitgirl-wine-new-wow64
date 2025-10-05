# Running Sims 3 FitGirl Repack installer under "new WoW64" Wine

My distro of choice, Arch Linux, transitioned its `wine` package to "new WoW64" in [June 2025](https://archlinux.org/news/transition-to-the-new-wow64-wine-and-wine-staging/). Most of my prefixes before that were 32-bit so I needed to recreate them. One of those prefixes contained a Sims 3 install and I finally looked into that last weekend.

Information in here was valid at the time of writing, i.e. early October 2025 and Wine 10.16.

Unfortunately there's no `tl;dr` here as the process is a bit involved.

## Fixing unarc.dll

One of the DLLs that the FitGirl installer uses contains a bug in the code that's triggered when running under "new WoW64" due to its `VirtualAlloc` never reporting an error for the parameters it's called with, which causes the code to go into an infinite loop.

This can be fixed by making the following changes to `unarc.dll` itself :

```diff
 6108a933:	89 34 24             	mov    %esi,(%esp)
 6108a936:	e8 5d 01 00 00       	call   0x6108aa98
 6108a93b:	83 3e 00             	cmpl   $0x0,(%esi)
-6108a93e:	74 04                	je     0x6108a944
-6108a940:	89 dd                	mov    %ebx,%ebp
-6108a942:	eb 02                	jmp    0x6108a946
-6108a944:	89 df                	mov    %ebx,%edi
+6108a93e:	75 0f                	jne    0x6108a94f
+6108a940:	89 df                	mov    %ebx,%edi
+6108a942:	90                   	nop
+6108a943:	90                   	nop
+6108a944:	90                   	nop
+6108a945:	90                   	nop
 6108a946:	89 f8                	mov    %edi,%eax
 6108a948:	29 e8                	sub    %ebp,%eax
 6108a94a:	83 f8 01             	cmp    $0x1,%eax
```

The file offset where the changes start is `9d3e`.

The loop that eventually becomes infinite is skipped on the first successful call to `VirtualAlloc`, which in case of "new WoW64" is the very first call.

## cls-precomp.dll crashes

The next problem I faced was `cls-precomp.dll`, another decompressor component used by the installer, constantly crashing with the following backtrace :

```
Unhandled exception: page fault on read access to 0x01171000 in 32-bit code (0x00413fcb).
Register dump:
 CS:0023 SS:002b DS:002b ES:002b FS:0063 GS:006b
 EIP:00413fcb ESP:00defa44 EBP:00defa78 EFLAGS:00010216(  R- --  I   -A-P- )
 EAX:01170ffc EBX:00defab4 ECX:00000251 EDX:00000000
 ESI:00defa60 EDI:00000020
Stack dump:
0x00defa44:  00000000 00000005 00defab4 0041eada
0x00defa54:  00defb00 00defab4 01165ac8 01170fa8
0x00defa64:  01170ffc 0117124d 00000000 ab31899e
0x00defa74:  00000020 00defa94 0041e91f 00000005
0x00defa84:  00000005 00000000 01165ac8 00defb00
0x00defa94:  00defae0 0041ed57 00defab4 01165ac8
Backtrace:
=>0 0x00413fcb in cls-precomp (+0x13fcb) (0x00defa78)
  1 0x0041e91f in cls-precomp (+0x1e91f) (0x00defa94)
  2 0x0041ed57 in cls-precomp (+0x1ed57) (0x00defae0)
  3 0x00407a24 in cls-precomp (+0x7a24) (0x00defddc)
  4 0x00408280 in cls-precomp (+0x8280) (0x00deff08)
  5 0x00425860 in cls-precomp (+0x25860) (0x00deff50)
  6 0x7bb3fa58 in kernel32 (+0xfa58) (0x00deff68)
  7 0x7bcce127 in ntdll (+0xe127) (0x00deff80)
  8 0x7bd046e5 in ntdll (+0x446e5) (0x00deffec)
0x00413fcb cls-precomp+0x13fcb: movzbl 4(%eax), %edi
```

Interestingly, the crash occurred with "new WoW64", "old WoW64", and 32-bit prefixes. Things work fine on Windows.

Despite the name, this isn't a DLL but a normal executable that's launched by the installer to process `sns` files by executing it like `cls-precomp -E 001.sns -o 001_.sns`. I could reproduce the crash reliably by just running the executable and one of the `sns` files extracted from one of the temp directories left behind by the installer when it crashed.

I've never tracked down the cause of the bug but setting `WINEDEBUG=warn+heap` somehow helps.

## Putting it all together

This assumes the following :
 * `$WINEPREFIX` is the prefix location that you're installing into,
 * `$SOURCEDIR` is the directory containing the repack files (`setup.exe` and all the `.bin`s),
 * `$USER` is the name of the user running Wine.

1. Launch a terminal window, set your `WINEPREFIX`, go to `$SOURCEDIR` and `wine setup.exe`.
2. Click through the installer, make sure to uncheck *Update DirectX* and the *Download* options at the bottom of the component list. Once you get to the window with the progress bar, it will get stuck at 0%.
3. In another terminal window, do `WINEPREFIX=/your/wine/prefix wineserver -k -w` to forcefully kill the installer so it doesn't have a chance to clean up its temporary files. Close this window once the installer is killed. Remove the directory you chose to install Sims 3 to in the installer as otherwise the next (potentially successful) run will fail.
4. Go to `${WINEPREFIX}/drive_c/users/${USER}/AppData/Local/Temp/`. You should find two directories in there. Copy `setup.tmp` from one of them to your `$SOURCEDIR` and copy `unarc.dll` from the other one to somewhere else.
5. Delete those two directories in `Temp`.
6. Modify `unarc.dll` as instructed above.
7. In the first terminal window, `export WINEDEBUG=warn+heap`, and launch `winedbg setup.tmp '/SL5=$2009C,6321655,140800,'"$(winepath --windows "${SOURCEDIR}")\setup.exe"`.
8. Tell the debugger to break when loading DLLs : `break LoadLibraryA` and `break LoadLibraryW`.
9. Have the debugger continue by issuing `c`, followed by enter. Keep continuing until the installer window opens. Set the options to the ones you actually want this time.
10. Before clicking `Install` in the last window, open `${WINEPREFIX}/drive_c/users/${USER}/AppData/Local/Temp/`. There should be only one directory in there, enter it.
11. Click `Install`. This will cause the debugger to break again. Keep doing `continue` until `unarc.dll` appears in the temporary directory.
12. Copy the modified `unarc.dll` over the original one, delete the breakpoints (`delete 1` and `delete 2`) and then `continue`. The installer will now run.

That's it. The installation took about 20 minutes on my Ryzen 5950X. You can run the game by going to `Game/Bin` in the installation directory and doing `wine TS3W.exe`. I also installed dxvk via winetricks but this isn't necessary.

If you want to have *all the content* (some of it is provided as `Sims3Pack` files which must be actually "installed" into the game first), you'll need to use the launcher which is incredibly flaky with Wine and requires `dotnet20` and `vcrun2005sp1` to be installed via winetricks. Once you have that, do `wine Sims3LauncherW.exe`, click *Downloads*, then *Select All* and *Install*. Another small window with a progress bar should pop up, wait until it completes and close the launcher.
