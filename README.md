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

Firstly, download debug symbols for Wine. Starting with version 10.20-2, the Arch package for Wine does not contain debug symbols which are provided in a separate package (which is generally a good thing as only a handful of people are going to need those and it makes the core package about 1GiB smaller) which we're going to need as trying to break in `winedbg` will not work without them. At the time of writing, the only mirror providing `-debug` packages is [`geo.mirror.pkgbuild.com/`](https://geo.mirror.pkgbuild.com/extra-debug/os/x86_64/). You can paste the link to the package straight into pacman, for example `sudo pacman -U https://geo.mirror.pkgbuild.com/extra-debug/os/x86_64/wine-debug-11.1-2-x86_64.pkg.tar.zst`.

You can uninstall the debug symbols after completing the installation as they won't be needed anymore.

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

## TS3W.exe crashes with a "Division by zero" error

`tl;dr` Change the byte in `TS3W.exe` at offset `0x21211c` from `F8` to `E8`.

This will happen only with some CPUs. This is the code that triggers this :
```
0x0061210a      mov eax, 4
0x0061210f      xor ecx, ecx
0x00612111      cpuid
0x00612113      mov dword [var_8h], eax
0x00612117      mov eax, dword [var_8h]
0x0061211b      sar eax, 26
0x0061211e      pop esi
0x0061211f      add eax, 1
// rest of epilogue
```

The value returned from this function (i.e. the value in `eax`) is then used as the divisor for something else. The value the code is after is contained in the six upper bits of EAX after a CPUID(EAX=4,ECX=0) which is supposed to be the *Maximum number of addressable IDs for processor cores in physical package, minus 1*.

However, if that value is the maximum allowed value, i.e. 63 or `0b111111`, then using `sar` to shift it to the right will cause `eax` to end up with an all-one bit pattern, i.e. `-1` in two's complement, due to `sar` copying the uppermost bit. Executing the `add eax, 1` later will result in setting it to zero. Replacing `sar` with `shr` (as noted in the `tl;dr`) fixes this, and the origin of this bug is probably an `int` being used instead of an `unsigned int` when doing the extractions in the original C code.

The values found in this field vary wildly between CPUs :
 * Both of my AMD CPUs (5950X and 5800X3D) return zeroes in all registers after a CPUID(EAX=4,ECX=0).
 * Intel Core i7-3632QM (Ivy Bridge, 3rd gen) sets the field in question to 7 which makes sense as it's a 8-thread CPU.
 * Intel Celeron J4105 (Gemini Lake / Goldmont Plus) sets it to 31, so the code wouldn't crash, though the CPU is also a 4-thread one so the value is peculiar.
 * Any newer Intel CPUs I tested with (N100 / Alder Lake-N, i9-13950HX / Raptor Lake 13th gen, Ultra 5 225U / Arrow Lake) set it to 63 regardless of the number of threads/cores in the CPU itself.

It seems like the meaning of this field must have changed somehow (unless there's something I don't know and it is possible to have 64 *addressable IDs for processor cores  in physical package* in a 32- or 4-thread CPU indeed, or those two quantities are simply unrelated despite the naming suggesting so) but Intel's Instruction Set Reference doesn't list 63 as a special value.
