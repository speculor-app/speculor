# Third-Party Notices

The Speculor binary distribution incorporates or ships alongside the following
third-party software. Each component listed below is governed by its own
license terms. Nothing in `LICENSE` or in any agreement You enter into with
Fabio Barros modifies those terms, and to the extent of any conflict, the
relevant third-party license prevails with respect to that component.

For the avoidance of doubt: Speculor uses an LGPL-only build of FFmpeg
(no `--enable-gpl`, no `libx264`, no `libx265`, no `--enable-nonfree`) and
dynamically links Qt 6 in a manner consistent with the LGPL.

## Summary

| Component | Version | License | SPDX |
|---|---|---|---|
| Qt 6 | 6.x | GNU Lesser General Public License v3.0 (with Qt LGPL Exception) | LGPL-3.0-only |
| QtNodes (`paceholder/nodeeditor`) | git 0d28130bd0a3 | BSD 3-Clause License | BSD-3-Clause |
| nlohmann/json | 3.11.3 | MIT License | MIT |
| FFmpeg | 7.1.1 | GNU Lesser General Public License v2.1 or later | LGPL-2.1-or-later |
| nv-codec-headers | n12.2.72.0 | MIT License | MIT |
| OpenCV | 4.x | Apache License 2.0 | Apache-2.0 |
| Eigen | 3.x | Mozilla Public License 2.0 | MPL-2.0 |
| libjpeg-turbo | 3.x | BSD 3-Clause + IJG + zlib | BSD-3-Clause |
| libcurl | recent | curl license (MIT/X derivative) | curl |
| Vulkan-Headers | recent | Apache License 2.0 OR MIT | Apache-2.0 OR MIT |
| Vulkan-Loader | recent | Apache License 2.0 | Apache-2.0 |
| glslang | recent | BSD 3-Clause / Apache License 2.0 (mixed) | BSD-3-Clause AND Apache-2.0 |
| oneTBB | recent | Apache License 2.0 | Apache-2.0 |
| libsodium | recent | ISC License | ISC |
| zlib | recent | zlib License | Zlib |

Full license texts for the longer licenses (LGPL-3.0, LGPL-2.1, Apache-2.0,
Mozilla Public License 2.0) appear in **Appendix A** at the end of this
document. Short license texts (BSD, MIT, ISC, zlib, curl) are reproduced
inline below.

---

## Qt 6

- Project: Qt Project / The Qt Company
- URL: https://www.qt.io/
- License: GNU Lesser General Public License v3.0 (LGPL-3.0), with the Qt
  LGPL Exception version 1.1 where applicable.
- License text: see Appendix A.

Speculor dynamically links the LGPL-licensed editions of the Qt 6 libraries
(at least `Qt6Core`, `Qt6Gui`, `Qt6Widgets`, `Qt6Network`, `Qt6OpenGL`).

In accordance with the LGPL, the corresponding source code for the version of
Qt shipped with this Speculor release is available from the Qt Project at
<https://download.qt.io/>. End users are entitled to modify the Qt libraries
distributed with Speculor and to replace the bundled Qt shared libraries with
their own modified or rebuilt versions; Speculor does not employ any technical
measure preventing such replacement.

Qt is a registered trademark of The Qt Company Ltd.

## QtNodes (paceholder/nodeeditor)

- Project: <https://github.com/paceholder/nodeeditor>
- Revision shipped: commit `0d28130bd0a3` (with patches applied at build time;
  patches are visible in `CMakeLists.txt`).
- License: BSD 3-Clause License.

```
BSD-3-Clause license
====================

Copyright (c) 2022, Dmitry Pinaev
All rights reserved.

Redistribution and use in source and binary forms, with or without modification,
are permitted provided that the following conditions are met:

  * Redistributions of source code must retain the above copyright notice, this
    list of conditions and the following disclaimer.
  * Redistributions in binary form must reproduce the above copyright notice,
    this list of conditions and the following disclaimer in the documentation
    and/or other materials provided with the distribution.
  * Neither the name of copyright holder, nor the names of its contributors may
    be used to endorse or promote products derived from this software without
    specific prior written permission.

THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS "AS IS" AND
ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE IMPLIED
WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE ARE
DISCLAIMED. IN NO EVENT SHALL THE COPYRIGHT OWNER OR CONTRIBUTORS BE LIABLE FOR
ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL, EXEMPLARY, OR CONSEQUENTIAL DAMAGES
(INCLUDING, BUT NOT LIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS OR SERVICES; LOSS
OF USE, DATA, OR PROFITS; OR BUSINESS INTERRUPTION) HOWEVER CAUSED AND ON ANY
THEORY OF LIABILITY, WHETHER IN CONTRACT, STRICT LIABILITY, OR TORT (INCLUDING
NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY OUT OF THE USE OF THIS SOFTWARE, EVEN
IF ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.
```

## nlohmann/json

- Project: <https://github.com/nlohmann/json>
- Version: 3.11.3
- License: MIT License.

```
MIT License

Copyright (c) 2013-2025 Niels Lohmann

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
```

## FFmpeg

- Project: <https://ffmpeg.org/>
- Version: 7.1.1
- License: GNU Lesser General Public License v2.1 or later (LGPL-2.1+).

Speculor builds and dynamically links an **LGPL-only** edition of FFmpeg:
the configure flags `--enable-gpl`, `--enable-libx264`, `--enable-libx265`,
and `--enable-nonfree` are deliberately **not** used. The shared libraries
`libavformat`, `libavcodec`, `libavdevice`, `libavfilter`, `libswscale`,
`libswresample`, and `libavutil` are redistributed under the LGPL.

License text: see Appendix A (LGPL-2.1).

In accordance with the LGPL, the corresponding source code for the version of
FFmpeg shipped with this Speculor release is available from
<https://ffmpeg.org/releases/> (file `ffmpeg-7.1.1.tar.xz`, SHA-256
`733984395e0dbbe5c046abda2dc49a5544e7e0e1e2366bba849222ae9e3a03b1`). The
FFmpeg configure flags Speculor uses are reproduced verbatim in the file
`cmake/deps/FFmpeg.cmake` of the Speculor SDK source tree.

FFmpeg is the trademark of Fabrice Bellard, originator of the FFmpeg project.

### nv-codec-headers (transitively used by FFmpeg's NVENC/NVDEC wrappers)

- Project: <https://github.com/FFmpeg/nv-codec-headers>
- Version: n12.2.72.0
- License: MIT License.

```
nv-codec-headers
Copyright (c) 2008-2024 NVIDIA Corporation

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in
all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN
THE SOFTWARE.
```

## OpenCV

- Project: <https://opencv.org/>
- Version: 4.x (see `vcpkg.json` for the exact baseline)
- License: Apache License 2.0.

License text: see Appendix A (Apache-2.0).

Copyright notices are reproduced from the OpenCV upstream `LICENSE` file and
preserved in the OpenCV binaries shipped with Speculor.

## Eigen

- Project: <https://eigen.tuxfamily.org/>
- Version: 3.x
- License: Mozilla Public License 2.0 (MPL-2.0).

License text: see Appendix A (MPL-2.0).

Eigen contains a small number of files under other permissive licenses
(BSD 3-Clause, MINPACK); those files are not shipped in the Speculor binary
distribution. See the Eigen `COPYING.README` file for details.

## libjpeg-turbo

- Project: <https://libjpeg-turbo.org/>
- Version: 3.x
- License: BSD 3-Clause License (modified) plus IJG and zlib licenses for
  certain components.

The libjpeg-turbo license requires the following acknowledgment in
documentation accompanying binary distributions:

> This software is based in part on the work of the Independent JPEG Group.

Full license text accompanies the libjpeg-turbo source distribution at
<https://github.com/libjpeg-turbo/libjpeg-turbo/blob/main/LICENSE.md>.

## libcurl

- Project: <https://curl.se/libcurl/>
- License: curl License (MIT/X derivative).

```
COPYRIGHT AND PERMISSION NOTICE

Copyright (c) 1996 - 2026, Daniel Stenberg, <daniel@haxx.se>, and many
contributors, see the THANKS file.

All rights reserved.

Permission to use, copy, modify, and distribute this software for any purpose
with or without fee is hereby granted, provided that the above copyright
notice and this permission notice appear in all copies.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT OF THIRD PARTY RIGHTS. IN
NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM,
DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR
OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE
OR OTHER DEALINGS IN THE SOFTWARE.

Except as contained in this notice, the name of a copyright holder shall not
be used in advertising or otherwise to promote the sale, use or other dealings
in this Software without prior written authorization of the copyright holder.
```

## Vulkan-Headers

- Project: <https://github.com/KhronosGroup/Vulkan-Headers>
- License: Apache License 2.0 OR MIT (dual-licensed at recipient's choice).

License text: see Appendix A (Apache-2.0). The MIT alternative is reproduced
in the upstream `LICENSE.md`. Speculor exercises the Apache 2.0 option.

## Vulkan-Loader

- Project: <https://github.com/KhronosGroup/Vulkan-Loader>
- License: Apache License 2.0.

License text: see Appendix A (Apache-2.0).

## glslang

- Project: <https://github.com/KhronosGroup/glslang>
- License: Mixed (BSD 3-Clause, Apache License 2.0, NVIDIA portions under
  permissive terms). The combined license is BSD 3-Clause AND Apache-2.0.

The upstream `LICENSE.txt` reproduces all required notices and is preserved
in the SPIR-V compilation tooling shipped with Speculor.

## oneTBB

- Project: <https://github.com/oneapi-src/oneTBB>
- License: Apache License 2.0.

License text: see Appendix A (Apache-2.0).

## libsodium

- Project: <https://libsodium.org/>
- License: ISC License.

```
ISC License

Copyright (c) 2013-2026
Frank Denis <j at pureftpd dot org>

Permission to use, copy, modify, and/or distribute this software for any
purpose with or without fee is hereby granted, provided that the above
copyright notice and this permission notice appear in all copies.

THE SOFTWARE IS PROVIDED "AS IS" AND THE AUTHOR DISCLAIMS ALL WARRANTIES
WITH REGARD TO THIS SOFTWARE INCLUDING ALL IMPLIED WARRANTIES OF
MERCHANTABILITY AND FITNESS. IN NO EVENT SHALL THE AUTHOR BE LIABLE FOR ANY
SPECIAL, DIRECT, INDIRECT, OR CONSEQUENTIAL DAMAGES OR ANY DAMAGES
WHATSOEVER RESULTING FROM LOSS OF USE, DATA OR PROFITS, WHETHER IN AN
ACTION OF CONTRACT, NEGLIGENCE OR OTHER TORTIOUS ACTION, ARISING OUT OF OR
IN CONNECTION WITH THE USE OR PERFORMANCE OF THIS SOFTWARE.
```

## zlib (transitively used by libcurl)

- Project: <https://zlib.net/>
- License: zlib License.

```
(C) 1995-2024 Jean-loup Gailly and Mark Adler

This software is provided 'as-is', without any express or implied
warranty.  In no event will the authors be held liable for any damages
arising from the use of this software.

Permission is granted to anyone to use this software for any purpose,
including commercial applications, and to alter it and redistribute it
freely, subject to the following restrictions:

1. The origin of this software must not be misrepresented; you must not
   claim that you wrote the original software. If you use this software
   in a product, an acknowledgment in the product documentation would be
   appreciated but is not required.
2. Altered source versions must be plainly marked as such, and must not be
   misrepresented as being the original software.
3. This notice may not be removed or altered from any source distribution.

Jean-loup Gailly        Mark Adler
jloup@gzip.org          madler@alumni.caltech.edu
```

---

# Appendix A: Full license texts

## A.1 GNU Lesser General Public License v3.0 (LGPL-3.0)

The LGPL-3.0 incorporates the GNU General Public License v3.0 by reference.
Both texts are available at:

- LGPL-3.0: <https://www.gnu.org/licenses/lgpl-3.0.txt>
- GPL-3.0:  <https://www.gnu.org/licenses/gpl-3.0.txt>

The full text is reproduced verbatim in the Qt source distribution that
accompanies each Qt release.

## A.2 GNU Lesser General Public License v2.1 (LGPL-2.1)

The LGPL-2.1 incorporates the GNU General Public License v2 by reference.
Both texts are available at:

- LGPL-2.1: <https://www.gnu.org/licenses/old-licenses/lgpl-2.1.txt>
- GPL-2.0:  <https://www.gnu.org/licenses/old-licenses/gpl-2.0.txt>

The full text accompanies the FFmpeg source distribution
(`COPYING.LGPLv2.1` in the FFmpeg source tarball referenced above).

## A.3 Apache License 2.0

```
                                 Apache License
                           Version 2.0, January 2004
                        http://www.apache.org/licenses/

   TERMS AND CONDITIONS FOR USE, REPRODUCTION, AND DISTRIBUTION

   1. Definitions.

      "License" shall mean the terms and conditions for use, reproduction,
      and distribution as defined by Sections 1 through 9 of this document.

      "Licensor" shall mean the copyright owner or entity authorized by
      the copyright owner that is granting the License.

      "Legal Entity" shall mean the union of the acting entity and all
      other entities that control, are controlled by, or are under common
      control with that entity. For the purposes of this definition,
      "control" means (i) the power, direct or indirect, to cause the
      direction or management of such entity, whether by contract or
      otherwise, or (ii) ownership of fifty percent (50%) or more of the
      outstanding shares, or (iii) beneficial ownership of such entity.

      "You" (or "Your") shall mean an individual or Legal Entity
      exercising permissions granted by this License.

      "Source" form shall mean the preferred form for making modifications,
      including but not limited to software source code, documentation
      source, and configuration files.

      "Object" form shall mean any form resulting from mechanical
      transformation or translation of a Source form, including but
      not limited to compiled object code, generated documentation,
      and conversions to other media types.

      "Work" shall mean the work of authorship, whether in Source or
      Object form, made available under the License, as indicated by a
      copyright notice that is included in or attached to the work
      (an example is provided in the Appendix below).

      "Derivative Works" shall mean any work, whether in Source or Object
      form, that is based on (or derived from) the Work and for which the
      editorial revisions, annotations, elaborations, or other modifications
      represent, as a whole, an original work of authorship. For the purposes
      of this License, Derivative Works shall not include works that remain
      separable from, or merely link (or bind by name) to the interfaces of,
      the Work and Derivative Works thereof.

      "Contribution" shall mean any work of authorship, including
      the original version of the Work and any modifications or additions
      to that Work or Derivative Works thereof, that is intentionally
      submitted to Licensor for inclusion in the Work by the copyright owner
      or by an individual or Legal Entity authorized to submit on behalf of
      the copyright owner. For the purposes of this definition, "submitted"
      means any form of electronic, verbal, or written communication sent
      to the Licensor or its representatives, including but not limited to
      communication on electronic mailing lists, source code control systems,
      and issue tracking systems that are managed by, or on behalf of, the
      Licensor for the purpose of discussing and improving the Work, but
      excluding communication that is conspicuously marked or otherwise
      designated in writing by the copyright owner as "Not a Contribution."

      "Contributor" shall mean Licensor and any individual or Legal Entity
      on behalf of whom a Contribution has been received by Licensor and
      subsequently incorporated within the Work.

   2. Grant of Copyright License. Subject to the terms and conditions of
      this License, each Contributor hereby grants to You a perpetual,
      worldwide, non-exclusive, no-charge, royalty-free, irrevocable
      copyright license to reproduce, prepare Derivative Works of,
      publicly display, publicly perform, sublicense, and distribute the
      Work and such Derivative Works in Source or Object form.

   3. Grant of Patent License. Subject to the terms and conditions of
      this License, each Contributor hereby grants to You a perpetual,
      worldwide, non-exclusive, no-charge, royalty-free, irrevocable
      (except as stated in this section) patent license to make, have made,
      use, offer to sell, sell, import, and otherwise transfer the Work,
      where such license applies only to those patent claims licensable
      by such Contributor that are necessarily infringed by their
      Contribution(s) alone or by combination of their Contribution(s)
      with the Work to which such Contribution(s) was submitted. If You
      institute patent litigation against any entity (including a
      cross-claim or counterclaim in a lawsuit) alleging that the Work
      or a Contribution incorporated within the Work constitutes direct
      or contributory patent infringement, then any patent licenses
      granted to You under this License for that Work shall terminate
      as of the date such litigation is filed.

   4. Redistribution. You may reproduce and distribute copies of the
      Work or Derivative Works thereof in any medium, with or without
      modifications, and in Source or Object form, provided that You
      meet the following conditions:

      (a) You must give any other recipients of the Work or
          Derivative Works a copy of this License; and

      (b) You must cause any modified files to carry prominent notices
          stating that You changed the files; and

      (c) You must retain, in the Source form of any Derivative Works
          that You distribute, all copyright, patent, trademark, and
          attribution notices from the Source form of the Work,
          excluding those notices that do not pertain to any part of
          the Derivative Works; and

      (d) If the Work includes a "NOTICE" text file as part of its
          distribution, then any Derivative Works that You distribute must
          include a readable copy of the attribution notices contained
          within such NOTICE file, excluding those notices that do not
          pertain to any part of the Derivative Works, in at least one
          of the following places: within a NOTICE text file distributed
          as part of the Derivative Works; within the Source form or
          documentation, if provided along with the Derivative Works; or,
          within a display generated by the Derivative Works, if and
          wherever such third-party notices normally appear. The contents
          of the NOTICE file are for informational purposes only and
          do not modify the License. You may add Your own attribution
          notices within Derivative Works that You distribute, alongside
          or as an addendum to the NOTICE text from the Work, provided
          that such additional attribution notices cannot be construed
          as modifying the License.

      You may add Your own copyright statement to Your modifications and
      may provide additional or different license terms and conditions
      for use, reproduction, or distribution of Your modifications, or
      for any such Derivative Works as a whole, provided Your use,
      reproduction, and distribution of the Work otherwise complies with
      the conditions stated in this License.

   5. Submission of Contributions. Unless You explicitly state otherwise,
      any Contribution intentionally submitted for inclusion in the Work
      by You to the Licensor shall be under the terms and conditions of
      this License, without any additional terms or conditions.
      Notwithstanding the above, nothing herein shall supersede or modify
      the terms of any separate license agreement you may have executed
      with Licensor regarding such Contributions.

   6. Trademarks. This License does not grant permission to use the trade
      names, trademarks, service marks, or product names of the Licensor,
      except as required for describing the origin of the Work and
      reproducing the content of the NOTICE file.

   7. Disclaimer of Warranty. Unless required by applicable law or
      agreed to in writing, Licensor provides the Work (and each
      Contributor provides its Contributions) on an "AS IS" BASIS,
      WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or
      implied, including, without limitation, any warranties or conditions
      of TITLE, NON-INFRINGEMENT, MERCHANTABILITY, or FITNESS FOR A
      PARTICULAR PURPOSE. You are solely responsible for determining the
      appropriateness of using or redistributing the Work and assume any
      risks associated with Your exercise of permissions under this License.

   8. Limitation of Liability. In no event and under no legal theory,
      whether in tort (including negligence), contract, or otherwise,
      unless required by applicable law (such as deliberate and grossly
      negligent acts) or agreed to in writing, shall any Contributor be
      liable to You for damages, including any direct, indirect, special,
      incidental, or consequential damages of any character arising as a
      result of this License or out of the use or inability to use the
      Work (including but not limited to damages for loss of goodwill,
      work stoppage, computer failure or malfunction, or any and all
      other commercial damages or losses), even if such Contributor
      has been advised of the possibility of such damages.

   9. Accepting Warranty or Additional Liability. While redistributing
      the Work or Derivative Works thereof, You may choose to offer,
      and charge a fee for, acceptance of support, warranty, indemnity,
      or other liability obligations and/or rights consistent with this
      License. However, in accepting such obligations, You may act only
      on Your own behalf and on Your sole responsibility, not on behalf
      of any other Contributor, and only if You agree to indemnify,
      defend, and hold each Contributor harmless for any liability
      incurred by, or claims asserted against, such Contributor by reason
      of your accepting any such warranty or additional liability.

   END OF TERMS AND CONDITIONS
```

## A.4 Mozilla Public License 2.0 (MPL-2.0)

The full text of the MPL-2.0 is available at:

- <https://www.mozilla.org/en-US/MPL/2.0/>

The MPL is a file-level weak copyleft license: only files released under the
MPL-2.0 (and modifications to those files) are subject to MPL terms. Speculor
does not modify any MPL-licensed source file from Eigen.

---

# Reporting

If You believe a third-party component is incorporated into Speculor without
appropriate attribution here, or if You spot an error in an attribution
above, please open an issue or contact the maintainer at
<https://github.com/speculor-app/speculor-app/issues>.
