![pgp-key-generation CI](https://github.com/summitto/pgp-key-generation/workflows/pgp-key-generation%20CI/badge.svg)

# PGP key generation

This repository provides the source for a utility used for creating PGP keys using
[libsodium](https://download.libsodium.org/doc/ "Introduction - Libsodium documentation")
(for ed- and curve25519 keys) or
[Crypto++](https://www.cryptopp.com/ "Crypto++ Library | Free C++ Class Library of Cryptographic Schemes")
(for nist-p256 and RSA keys).

Generating keys will also output a 64-character hexadecimal seed value, which, in
combination with the chosen passphrase, can be used to recreate the exact same key.

This means that keeping the seed value and passphrase in a safe location can protect
you against losing keys, since they can always be generated again.

**Although an audit has been completed (see [below](#audit)), use this security-related
tool at your own risk!**

## Dependencies

The source code can be built using only the dependencies of the
[pgp-packet-library](https://github.com/summitto/pgp-packet-library).
It is recommended to build this tool and the library with the same compilers.

The integration testing script, which can be run using `make test` in
the build folder, additionally requires Python 3.7 (or 3.6 with
the `dataclasses` library) and GnuPG to be installed.

The tool was compiled and tested with the following compilers
| Compiler      | Version         |
| :---          |     :---:       |
| Apple clang   | 10.0.3          |
| clang         | 6.0.0           |
| gcc           | 8.0.0           |

## Audit

This tool has been audited by [Radically Open
Security](https://radicallyopensecurity.com/) in November 2019:

    During this review we focused on weak key material being generated and
    sensitive data being leaked.  
    
    We found a couple of issues which would probably not cause any problems if
    the tool is used as intended. Operator errors can never be ruled out,
    however, so it makes sense to build defense in depth to limit the
    possibilities of such a thing happening.  
    
    We also found two cases of invalid or insecure parameters being used, which
    could lead to the choosing of weak key material or cryptographic
    algorithms.

[Read full audit report](https://github.com/summitto/pgp-key-generation/audit.pdf)


## Generating new keys

Please read all of the steps below thoroughly before actually starting to
generate a key.

- If you have a new smartcard, change the user and admin pin first. See:
  https://www.gnupg.org/howtos/card-howto/en/ch03s02.html
- install `GnuPG` and this utility on a secure, offline computer. See:
  https://github.com/summitto/raspbian_setup
- install `GnuPG` on your main device. Optionally, `scdaemon`, `libccid` and
  `pcscd` may need to be installed.
- raise the lockable memory limit to at least 1M in `/etc/security/limits.conf` and get a new session
- run key generation utility, for example using:
```
generate_derived_key -o keyfile -t eddsa -n "firstname lastname" -e email -s "2011-01-01 01:01:01" -x "2099-09-09 09:09:09" -k "12345678" -c "2011-01-01 01:01:01"
```
- The program will either generate a new encrypted seed using dice input, or you
  can use an existing encrypted seed to generate your key. If you generate a
  new seed, store it in a secure place.  
- import the generated key file into gpg with `gpg --import ${KEYFILE}`. 

If you have a smart card, you can import the private key as follows:
- insert the smart card (e.g. Yubikey or Nitrokey)
- run `gpg --key-edit keyid` and provide the next input
```
toggle
key 1
keytocard
key 1
key 2
keytocard
key 2
key 3
keytocard
save
```
- export your public key with `gpg --export ${KEYID} > ${KEYFILE}`
- delete the keys from gpg with `gpg --delete-secret-and-public-keys ${KEYID}`
- copy the public key file to a usb stick and import it on your target computer
- insert the smart card in the target computer
- run `gpg --card-edit` and `fetch`

You should now have a functional key. You can test it as follows:

- list all the keys with `gpg --list-keys` 
- create a file to encrypt with `echo helloworld > test.txt`
- encrypt the file with `gpg -r ${KEYID} --encrypt test.txt`
- decrypt the file with `gpg --decrypt test.txt.gpg`

## Updating existing keys

If you want to change the expiry date of existing keys, you can simply follow
the steps above again to generate a new key with a different expiry date, using
your encrypted seed and passphrase. 

Note that when you import your key into gpg, you should check whether the key
ID matches the key ID of your previous key. This ensures that you didn't make
any errors when passing data to the key generation utility.

Note that before importing a public key with a new expiry date into `GnuPG`,
you must delete your old public key first. 

## Upgrading hardware tokens

Recently a security bug was found for Nitrokey Start devices, which allows
extracting the private key from the device - the very thing it is meant to
protect against. If you are using one of these devices you should ensure that
you are using the latest firmware. Instructions can be found on the [Nitrokey
Start release
page](https://github.com/Nitrokey/nitrokey-start-firmware/releases).

Be aware that flashing firmware will erase keys currently residing on the
device.

## Warnings and limitations

To avoid private key data from leaking, whenever possible, secret data is
prevented from being swapped to disk. This is not guaranteed to work reliably
on all platforms, however so it is strongly recommended to completely disable
all swap partitions before using the tool.

## Static analysis

### CppCheck

The project can be analyzed with
[Cppcheck](http://cppcheck.sourceforge.net/) by using the `cppcheck`
target. This target is available only if the `cppcheck` binary can be
found.

### Clang Tidy

If the `clang-tidy` binary can be found, the `tidy` target will be available
for `make` to run the checks configured in `.clang-tidy`.

