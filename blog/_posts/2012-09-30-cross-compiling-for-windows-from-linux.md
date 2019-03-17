---
layout: post
title: Cross compiling Windows binaries from Linux
date: '2012-09-30T23:21:00.001-07:00'
author: David Coles
tags:
- Programming
- Windows
- C
- Porting
- MingGW
modified_time: '2019-03-17T00:21:00-07:00'
redirect_from:
  - /blog/2012/09/cross-compiling-for-windows-from-linux.html
---

Recently I was helping out a friend with some C/C++ programming work for one of
his classes. One issue was that his university uses Microsoft Visual Studio as
their compiler which causes a bit of a problem if you don't have that installed
and want to make sure a particular bit of code actually compiles.

Now I could have installed [Visual Studio Express Edition](https://visualstudio.microsoft.com/vs/express/)
(thankfully it seems Microsoft has [had a change of heart](http://blogs.msdn.com/b/visualstudio/archive/2012/09/12/visual-studio-express-2012-for-windows-desktop-is-here.aspx)
and decided that you can keep developing desktop apps with the latest 2012
release), though I have rather limited space on my Windows desktop as it's
mostly reserved for playing games 😁. Thus I decided to work out how hard it is
to cross compile Windows binaries from a Linux machine.

Thankfully, due to the existence of [MinGW-w64](http://mingw-w64.sourceforge.net/),
the answer turns out to be "not too hard". First off, on Debian based systems,
you'll require the [mingw-w64](apturl:mingw-w64) package to be installed. This
will install the C compiler as `i686-w64-mingw32-gcc` and `x86_64-w64-mingw32-gcc`
for 32-bit and 64-bit Windows binaries respectively.

Now since Windows isn't a Unix operating system, it doesn't have any of the
standard headers you might expect, instead MingGW-w64 provides a series of
Windows headers and libraries which can be found in either
`/usr/i686-w64-mingw32/` or `/usr/x86_64-w64-mingw32/`. These directories will
automatically be included by the compiler, so you shouldn't need to manually
specify the include or library directories with `-I` or `-L`.

From this point it should just be a case of running something like
`i686-w64-mingw32-gcc -o prog.exe prog.c -lwsock32` to get a binary you can
happily run on a Windows machine without any further problems.

Here's some hints I found while porting an program from Unix to the Win32 API:

- You can use `#ifdef _WIN32` to check if you're compiling for the Windows API
  (defined for both the 32-bit and 64-bit compilers).
- Include `winsock2.h` for access to the socket API. The POSIX headers don't
  exist, but most of the standard functions are available.
- Use [`WSAGetLastError`](http://msdn.microsoft.com/en-us/library/windows/desktop/ms741580(v=vs.85).aspx)
  instead of `errno` when checking why a socket function failed. While the
  `errno.h` header exists, `errno` will always be set to 0.
- You must call [`WSAStartup`](http://msdn.microsoft.com/en-us/library/windows/desktop/ms742213(v=vs.85).aspx)
  before you call any socket functions, otherwise they'll return an error and
  `WSAGetLastError` will return [`WSANOTINITIALISED`](http://msdn.microsoft.com/en-us/library/windows/desktop/ms740668(v=vs.85).aspx#WSANOTINITIALISED)
  (10093). At very least you must do something like:
  ```c
  // Initialize WinSock API
  WORD version = MAKEWORD(2, 2);
  WSADATA wsaData;
  int err = WSAStartup(version, &wsaData);
  if (err != 0) {
      // Fatal error
      fprintf(stderr, "ERROR: WSAStartup (%d)\n", WSAGetLastError());
      exit(1);
  }
  ```
- You'll need to use the [Windows Desktop API](http://msdn.microsoft.com/en-us/library/windows/desktop/hh447209(v=vs.85).aspx)
  for everything (no pthreads and such). MingGW-w64 does not provide a POSIX
  compatibility environmental like [Cygwin](http://www.cygwin.com/) or
  [Windows Subsystem for Linux (WSL)](https://docs.microsoft.com/en-us/windows/wsl/).
