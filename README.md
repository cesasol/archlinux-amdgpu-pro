# Arch Linux AMDGPU PRO

This project contains a generator for the [amdgpu-pro-installer](https://aur.archlinux.org/pkgbase/amdgpu-pro-installer) PKGBUILD.

This generator is needed because of the complexity of the distribution model of drivers. It handles packages dependencies (maps debian dependencies to arch linux alternatives), calculates sources hashes and so on.

## Prerequisites

`pip install python-debian` # used in concat_packages_extracted.py
`pacman -S dpkg-deb` # used in translate_deb_to_arch_dependency.sh
`pacman -S aptly` # used for mirroring repo
`yay -S debtap` # used in dependencies translation

## Steps to do when new version is released
1. Go to http://repo.radeon.com/amdgpu/ and see if there is a new version available.
2. Change versions in `versions` file.
   How to know the build id version? You can see it in `Packages` file of the release for distro. For example, here: http://repo.radeon.com/amdgpu/22.20.5/ubuntu/dists/jammy/proprietary/binary-amd64/
   Note: focal=20.04, jammy=22.04, noble=24.04. Table of versions: https://en.wikipedia.org/wiki/Ubuntu_version_history#Table_of_versions
3. Change pkgrel, url_ref in `gen-PKGBUILD.py` file.
4. Install aptly if not done yet.
5. Update the local mirror
   Note: focal=20.04, jammy=22.04, noble=24.04..
   Note that `latest` link sometimes is not actually latest version.
   ```
   #ver=latest
   #ver=22.20.5
   ver=5.4.1
   aptly -ignore-signatures mirror create agpro-$ver http://repo.radeon.com/amdgpu/$ver/ubuntu jammy proprietary
   aptly -ignore-signatures mirror update agpro-$ver

   aptly publish drop jammy
   aptly snapshot create snapshot-$(date +%F) from mirror agpro-$ver
   aptly publish --skip-signing snapshot snapshot-$(date +%F) # Or if not skipping signing, when it will ask a password, do not enter empty or it will fail
   ```

   Now in ~/.aptly/public/pool (not in ~/.aptly/pool/) there will be our packages.
6. Run `./unpack_all_deb_packages.sh`
7. Bring the "Packages" file to "Packages-extracted" with the following command:
   `python concat_packages_extracted.py`
   Do not forget to replace "jammy" after new distribution is released. Note: focal=20.04, jammy=22.04.

   That command does merging Packages files for 32 bit and 64 bit, and then it automatically removes duplicated entries.
   This is a workaround. See more info in the gen_packages_map.sh in the beginning comment.

   There also could be such way: <s>`aptly package show "Name (~ .*)" > Packages-extracted`</s>. For some reason, this method shows filenames without relative path. So cannot use that until inversigate how to fix.

   Also files can be seen here (convenience links):
   http://repo.radeon.com/amdgpu/latest/ubuntu/dists/focal/proprietary/binary-amd64/Packages
   http://repo.radeon.com/amdgpu/latest/ubuntu/dists/focal/proprietary/binary-i386/Packages
8. Run `./gen_packages_map.sh > packages_map.py`
   See differences with `git diff -w packages_map.py`
   If there are differences, then make adjustments to gen_packages_map.sh if needed. Especially, look for the new appeared packages (they will have empty comment) and removed packages. If there are new or removed packages, then use make_pkgbuild_pkgs_template.sh and edit gen-PKGBUILD.py
9. Run `sudo debtap -u`. Then run `./gen_replace_deps.sh > replace_deps.py`
    See differences with `git diff -w replace_deps.py`
    If there are differences, then make adjustments to gen_replace_deps.sh if needed.
10. Note: this step is boring, and I let myself to skip it. But just in case, I will leave this instruction here, as it could be interesting to check some day. Also, don't forget to fill in old versions in versions file.
    Run `./extract_transaction_scripts_and_triggers.sh`.
    Compare transaction scripts and triggers from current and previous driver version in opened meld window.
    If you see explicit version numbers in file names, add additional rename instructions in extract_transaction_scripts_and_triggers.sh. This will help you to compare contents of the files in meld.
    If there are changes, see if it is needed to convert them to pacman .install files or hooks.

11. Create a local repository (if not done yet) by adding the following to your /etc/pacman.conf:
     ```
     [amdgpu-dev]
     SigLevel = Optional TrustAll
     Server = File:///home/andrey/Development/archlinux-amdgpu-pro/ # edit path to your development directory. Do not keep this comment in the config.
     ```
12. Regenerate PKGBUILD with `./remake_all.sh`
13. If you notice empty license in PKGBUILD, add its hash to the licenses_hashes_map in gen-PKGBUILD.py
14. Install from local repository and test it.
15. Make a new commit to project repo (not the aur).
    Then commit a new tag.
     ```
     git tag $(source PKGBUILD; echo v${major}_${minor}-${pkgrel})
     git push origin $(source PKGBUILD; echo v${major}_${minor}-${pkgrel})
     ```
    Create a release on github pointing to the new tag.
16. Update AUR package: copy PKGBUILD, .SRCINFO and other files as needed.

## Contributing

Please run `reposetup.sh` prior to committing. This will install a git hook
that will automatically re-generate `PKGBUILD` and `.SRCINFO` as needed
before the commit completes.

Note that running `gen-PKGBUILD.py` requires `python-debian` package.
