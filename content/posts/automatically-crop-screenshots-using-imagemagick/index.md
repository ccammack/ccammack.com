---
title: "Automatically Crop Screenshots Using ImageMagick"
date: 2020-10-17T09:49:39-07:00
tags: ["ImageMagick", "Windows"]
---

I don't like editing images by hand, but occasionally find myself needing to cut down a full page screenshot to extract something of interest like a dialog box with an open menu.

<!--more-->

As it turns out, as long as the desktop follows a consistent layout, it's easy to write a reusable [ImageMagick](https://imagemagick.org/index.php) script to automatically crop full page images down to size.

In my own case, the desktop normally has a column of icons down the left side and an open console window on the right. If I need to capture an object, I move it to the middle of the screen before taking the screenshot, leaving some empty space around the outside edges.

{{< figure src="input.png" alt="Full Page Screenshot">}}

{{< highlight txt >}}
C:\Users\ccammack\Desktop
λ magick --version
Version: ImageMagick 7.0.10-29 Q16 x64 2020-09-05 http://www.imagemagick.org
Copyright: Copyright (C) 1999-2018 ImageMagick Studio LLC
License: http://www.imagemagick.org/script/license.php
Visual C++: 192729111
Features: Cipher DPC HDRI Modules OpenCL OpenMP(2.0)
Delegates (built-in): bzlib cairo flif freetype gslib heic jng jp2 jpeg lcms lqr lzma openexr pangocairo png ps raw rsvg tiff webp xml zlib
{{< /highlight >}}

---

To figure out the right amount to crop from each side for this desktop layout, run some tests using **magick convert** on the full screen *input* image and specify the **show:** parameter to immediately view the results of the each command without specifying an output file.

To remove the *left* side of the image, set the reference origin to the *upper-left* corner using **-gravity northwest** and then specify **-chop 4%x0%** to remove **4%** from the left and **0%** from the top. Experiment with the chop percentages to find the right amount for the desktop layout.

{{< highlight txt >}}
C:\Users\ccammack\Desktop
λ magick convert input.png -gravity northwest -chop 4%x0% show:
{{< /highlight >}}

{{< figure src="after-northwest-chop.png" alt="After Northwest Chop">}}

---

Similarly, to remove the *bottom* and *right* sides of the image, set the reference origin to the *bottom-right* corner using **-gravity southeast** and then specify **-chop 37%x4%** to remove an additional **37%** from the right and **4%** from the bottom.

{{< highlight txt >}}
C:\Users\ccammack\Desktop
λ magick convert input.png -gravity northwest -chop 4%x0% -gravity southeast -chop 37%x4% show:
{{< /highlight >}}

{{< figure src="after-southeast-chop.png" alt="After Southeast Chop">}}

---

This leaves a solid blue background around the ouside of the image, which can be removed by adding the **-trim** parameter to the end of the command.

{{< highlight txt >}}
C:\Users\ccammack\Desktop
λ magick convert input.png -gravity northwest -chop 4%x0% -gravity southeast -chop 37%x4% -trim show:
{{< /highlight >}}

{{< figure src="after-trim.png" alt="After Trim">}}

---

In my testing, the **-trim** parameter left a 4-pixel blue border around the image, so I removed it by adding **-shave 4x4** to shave off the remaining 4 pixels from each edge.

{{< highlight txt >}}
C:\Users\ccammack\Desktop
λ magick convert input.png -gravity northwest -chop 4%x0% -gravity southeast -chop 37%x4% -trim -shave 4x4 show:
{{< /highlight >}}

{{< figure src="after-shave.png" alt="After Shave">}}

---

Once the procedure works properly, wrap the command in a batch(+powershell) script so it can process multiple images with one call and append the word *cropped* to each output filename.

{{< highlight txt >}}
<#  :cmd header for PowerShell script
@   set dir=%~dp0
@   set ps1="%TMP%\%~n0-%RANDOM%-%RANDOM%-%RANDOM%-%RANDOM%.ps1"
@   copy /b /y "%~f0" %ps1% >nul
@   powershell -NoProfile -ExecutionPolicy Bypass -File %ps1% %*
@   del /f %ps1%
@   goto :eof
#>
foreach ($a in $args) {
	$output = ( (Get-Item $a).DirectoryName + "\" + (Get-Item $a).Basename + ".cropped" + (Get-Item $a).Extension )
	& magick convert $a -gravity northwest -chop 4%x0% -gravity southeast -chop 37%x4% -trim -shave 4x4 +repage $output
}
{{< /highlight >}}

This batch file works on the command line and can also serve as a drop target so files can be automatically cropped by dragging and dropping them onto the batch file.
