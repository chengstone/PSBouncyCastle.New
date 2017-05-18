PSBouncyCastle.New
==================

**PSBouncyCastle** is a PowerShell module that allows you to use the
crypto functionality from the [Legion of the BouncyCastle](http://www.bouncycastle.org/)
.NET libraries.

Currently it covers the X509 certificate functionality, in particular
allowing you to replace `makecert.exe` (from the Windows SDK) with
native PowerShell cmdlets.

This was written by *RLipscome* and forked to the current version in order to update the Crypto library.

Installation
--

	Set-Location (Join-Path (Split-Path $PROFILE) 'Modules')
	git clone https://github.com/LimpingNinja/PSBouncyCastle.New.git
	Import-Module PSBouncyCastle

*Note:* I'll get this listed on [PsGet](http://psget.net/) at some point.
