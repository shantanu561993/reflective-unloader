# Reflective Unloader
This is code that can be used within a PE file to allow it to reflectively
reconstruct itself in memory at runtime. The result is a byte for byte copy of
the original PE file. This can be combined with [Reflective DLL Injection][1] to
allow code to reconstruct itself after being loaded through an arbitrary means.

The original PE file will not be modified in memory, this code makes a new copy
of the unloaded target module.

## Usage

1. The build environment is Visual Studio 2017.
1. Add `ReflectiveUnloader.c \ ReflectiveUnloader.h` to the desired project.
   Once added, call `ReflectiveUnloader()` with a handle to the module to unload
   and reconstruct.
   * For an executable this could be `GetModuleHandle(NULL)`<sup>[1][2]</sup>
   * For a DLL this could be `hinstDLL` from `DllMain`<sup>[2][3]</sup>
1. After compiling the project, run `pe_patch.py` to patch in necessary data to
   the pe file. Without this step, the writable sections of the PE file will be
   corrupted in the unloaded copy. (See [below](#visual-studio-build-event) for
   how to automate this.)

### PE Patching
It's necessary to patch the PE file to get a perfect byte-for-byte copy when it
is reconstructed. The patching process creates a new `.restore` section where a
copy of all writable sections are backed up. When the `ReflectiveUnloader`
function is then called, it will process this extra section to restore the
original contents to the writable sections.

If the `.restore` section is not present, the unloader will simply skip this
step. This allows the unloader to perform the same task for arbitrary unpatched
PE files, however **any modifications to segments made at runtime will be
present in the unloaded PE file**.

#### Visual Studio Build Event
The `pe_patch.py` script can be executed automatically for every build using a
build event. Right click the project in Solution Explorer, then navigate to
`Configuration Properties > Build Events > Post Build Event` and adjust the
settings as follows:

| Setting Name | Setting Value                                                      |
|--------------|--------------------------------------------------------------------|
| Command Line | `python $(SolutionDir)pe_patch.py "$(TargetPath)" "$(TargetPath)"` |
| Description  | Patch in the .restore section                                      |
| Use In Build | Yes                                                                |

### API Reference
#### ReflectiveUnloader
```
PVOID ReflectiveUnloader(
  _In_  HINSTANCE hInstance,
  _Out_ PSIZE_T   pdwSize
);
```
*hInstance* \[in\]
> Handle to the module instance to unload from memory

*pdwSize* \[out\]
> The size of the returned PE image

**Return value**

If the function succeeds, a pointer to the unloaded PE image is returned. The
data at this address is then suitable for reuse for other purposes such as being
written to disk or injected into another process.

If the function fails, the return value is `NULL`.

#### ReflectiveUnloaderFree
```
VOID ReflectiveUnloaderFree(
  _In_ PVOID  pAddress,
  _In_ SIZE_T dwSize
);
```
*pAddress* \[in\]
> Pointer to the blob returned by ReflectiveUnloader

*dwSize* \[in\]
> Size of the blob returned by ReflectiveUnloader

## Proof of Concept
The proof of concept included in the project is the `Main.c` file. This can be
compiled into a `ReflectiveUnloader.dll` which is compartible with
[Reflective DLL Injection][1]. The resulting executable can then be injected
into an arbitrary process (assuming premissions and architecture constraints are
met) with the [inject.exe][4] utility. Take note of the hash of the DLL file
before proceeding. See the [releases page][5] for pre-built binaries.

Once the DLL is injected into a process, it will display a message box. This is
used to present the user with an opportunity to delete the original DLL from
disk. After the message box is closed, a new and identical copy will be written
to `%USERPROFILE%\\Desktop\\ReflectiveUnloader.dll`.

Finally the user can compare the hashes of the two files to determine that they
are identical.

## License
This project is released under the BSD 3-clause license, for more details see
the [LICENSE][license-url] file.

## Credits

 - Brandan Geise - coldfusion ([@coldfusion39](https://twitter.com/coldfusion39))
 - Spencer McIntyre - zeroSteiner ([@zeroSteiner](https://twitter.com/zeroSteiner))

[1]: https://github.com/stephenfewer/ReflectiveDLLInjection
[2]: https://msdn.microsoft.com/en-us/library/windows/desktop/ms683199(v=vs.85).aspx
[3]: https://msdn.microsoft.com/en-us/library/windows/desktop/ms682583(v=vs.85).aspx
[4]: https://github.com/stephenfewer/ReflectiveDLLInjection/tree/master/bin
[5]: https://github.com/zeroSteiner/reflective-unloader/releases
[license-url]: https://github.com/zeroSteiner/reflective-unloader/blob/master/LICENSE
